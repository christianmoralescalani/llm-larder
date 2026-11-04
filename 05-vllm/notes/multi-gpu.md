# Multi-GPU — tensor parallelism in practice

**Summary.** Splitting one model across GPUs is one flag: `--tensor-parallel-size N` (short `-tp N`). Each GPU holds 1/N of every layer's weights and they run in lockstep (NCCL all-reduce between layers). For Docker that's `--gpus all` + `--ipc=host` + the flag — nothing else. Pipeline parallelism (`-pp`) splits layers into stages (mostly for multi-node); data parallelism (`-dp`) runs N independent replicas. Rule of thumb: **TP within a node, PP across nodes, DP for throughput scaling**.

## Tensor parallel (single node)

```bash
# 70B AWQ on 2×24GB (the canonical example)
docker run -d --name llm --gpus all -p 8000:8000 --ipc=host \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai:latest \
  --model Qwen/Qwen2.5-72B-Instruct-AWQ \
  --tensor-parallel-size 2 \
  --quantization awq --dtype half \
  --max-model-len 32768
```

What happens:

- 72B AWQ ≈ 40 GB of weights → ~20 GB per GPU; KV cache is also split.
- Each forward pass: both GPUs compute their half of each layer, then sync (all-reduce). Inter-GPU bandwidth (PCIe/NVLink) is the cost — NVLink makes TP nearly free, PCIe still works, multi-node TP is where it gets painful.
- Per-token latency ≈ single-GPU latency (that's the point of TP); **throughput scales** with the GPUs.

## Sizing: does it fit?

Quick math (02-hardware-and-vram has the full version):

```
VRAM needed ≈ weights + KV cache + activations/overhead
weights (AWQ 4-bit)  ≈ params × 0.5 bytes   (72B ≈ 36–40 GB)
KV per token (Llama-70B-ish) ≈ 2 × layers × kv_heads × head_dim × dtype_bytes
```

- 72B-AWQ + 32k ctx: ~20 GB/GPU on 2×24GB → fits with room for KV.
- 72B-AWQ + 128k ctx: KV dominates → `--kv-cache-dtype fp8` or drop `--max-model-len`.
- If it fits on **one** GPU, don't TP — the all-reduce overhead costs you a few percent.

## Data parallel (replicas)

Model fits on one GPU (or one TP group) but you want more throughput:

```bash
# 2 independent 8B replicas behind one endpoint
vllm serve Qwen/Qwen2.5-7B-Instruct --data-parallel-size 2
# or: 1 TP group × 2 replicas
vllm serve <big-model> --tensor-parallel-size 2 --data-parallel-size 2
```

- Requests are load-balanced across replicas; each replica has its own KV cache and prefix cache.
- Non-MoE: plain replicas. MoE: `--data-parallel-rank` enables external-LB mode where MoE layers share experts across the DP group.

## Expert parallel (MoE)

```bash
vllm serve <MoE-model> --tensor-parallel-size 2 --enable-expert-parallel
```

- Experts are sharded across GPUs instead of replicated within the TP group — matters when the expert count is large relative to the dense part.
- `--enable-eplb` + `--eplb-config`: dynamic expert load balancing (hot experts move).

## Pipeline parallel & multi-node

TP within a node, PP across nodes:

```bash
# node 0 (2 GPUs)
vllm serve <model> --tensor-parallel-size 2 --pipeline-parallel-size 2 \
  --nnodes 2 --node-rank 0 --master-addr 10.0.0.1 --master-port 29501
# node 1 (2 GPUs)
vllm serve <model> --tensor-parallel-size 2 --pipeline-parallel-size 2 \
  --nnodes 2 --node-rank 1 --master-addr 10.0.0.1 --master-port 29501
```

- `TP = GPUs per node`, `PP = number of nodes` (for the standard layout).
- PP adds pipeline bubbles (stages wait for each other); TP is preferred when the interconnect allows.
- Multi-node needs the nodes to reach each other on the NCCL ports (firewall!) and matching `--master-port`.
- In Docker, each node runs its own container with the same flags except `--node-rank`.

## Decision tree

```
model fits on 1 GPU?
├── yes → single instance; scale with --data-parallel-size (or more containers + LB)
└── no → fits on 1 node?
    ├── yes → --tensor-parallel-size N (N = GPUs on node)
    └── no  → -tp <gpus/node> -pp <nodes> + --nnodes/--node-rank
MoE at any size? → consider --enable-expert-parallel (with TP)
```

## Gotchas

1. **`--ipc=host` or it crashes** — TP workers share memory; without it you get `Bus error`/shared-memory failures.
2. **TP must divide the model** — `tensor_parallel_size` has to divide hidden size / attention heads; 2, 4, 8 are the safe values.
3. **PCIe vs NVLink** — TP over PCIe works but sync overhead grows with layer count; if you're comparing 1×80GB vs 2×40GB, benchmark, don't assume.
4. **KV cache is split too** — with TP the KV per request is N-way split, so long contexts fit better than the per-GPU math suggests.
5. **Multi-node = ops burden** — NCCL env (`NCCL_DEBUG=INFO` for debugging), firewall, matching driver/CUDA versions across nodes.
6. **One model per instance** — TP is *one model, many GPUs*; to serve several models use separate instances/replicas (or DP).
7. **`--max-num-seqs` is per-instance** — DP doesn't change it per replica; total concurrency = replicas × per-replica limit.

## Sources

- [Parallelism & scaling](https://docs.vllm.ai/en/stable/serving/parallelism_scaling/)
- [Data parallel deployment](https://docs.vllm.ai/en/stable/serving/data_parallel_deployment/)
- [Expert parallel deployment](https://docs.vllm.ai/en/stable/serving/expert_parallel_deployment/)
- [Multi-node serving example](https://docs.vllm.ai/en/stable/examples/ray_serving/multi-node-serving/)
- [vllm serve CLI reference (ParallelConfig)](https://docs.vllm.ai/en/stable/cli/serve/)

## My experiments

- (fill in: e.g. "72B-AWQ on 2×24GB with TP=2: cold start X min, decode Y t/s @ 1 req, Z t/s aggregate @ 16 concurrent; dropped `--max-model-len` to 32k to fit KV; DP=2 of 8B beat TP=2 of 8B for throughput as expected")
