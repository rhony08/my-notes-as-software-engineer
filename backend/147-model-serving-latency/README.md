# Serving Models: Latency, Batching, and GPUs

Your RAG feature works great in a notebook. Then you put it in front of 50 concurrent users and the p95 latency goes from 800ms to 9 seconds. Nothing is broken — you've just hit the reality of model serving: a GPU is a shared resource, and how you schedule work onto it decides whether your app feels snappy or stalls.

This is the part of AI engineering that looks like plain backend work. It is. You're managing throughput, queues, memory, and tail latency — just with a much more expensive, much more awkward worker behind the curtain.

## The Mental Model: A GPU Is a Batch Machine

A CPU core chews through one instruction stream fast. A GPU has thousands of small cores that want to do the *same* operation on *different* data at the same time. That's why it's great at matrix math — and why it's terrible at doing one tiny thing at a time.

An LLM forward pass is mostly giant matrix multiplies. Feeding it a single prompt leaves most of the chip idle, waiting on memory access. Feeding it 32 prompts at once barely costs more time than one, because the expensive part is memory bandwidth, not compute.

That gap between "one request" and "one batch" is where all the interesting engineering lives.

```
One request, unbatched:   [ GPU mostly idle ]  → 120ms
32 requests, batched:     [ GPU saturated   ]  → 140ms
```

Same hardware. ~28x the throughput for ~16% more latency. That's the whole game.

## Why Latency and Throughput Fight Each Other

You can't maximize both. Batching trades latency for throughput:

- **Batch size 1** → lowest latency per request, worst throughput, GPU sits idle.
- **Large batches** → best throughput, but every request waits for the rest of the batch to fill.

For an interactive chatbot, you want *low* latency, so batching is tempting to skip. For a nightly classification job, you want throughput and nobody cares if a request waits two seconds.

❌ The classic mistake: tuning one number for both workloads.

✅ Decide per-endpoint what you're optimizing for, then set batching policy accordingly.

| Workload | Optimize for | Batching policy |
|----------|-------------|-----------------|
| Chat / copilot | Latency | Small max batch, short wait window |
| Offline classification | Throughput | Large batches, no wait |
| Embedding ingestion | Throughput | Huge batches, pipelined |
| Interactive autocomplete | Latency (hard) | Batch size 1, accept idle GPU |

## Dynamic Batching: Waiting Just Long Enough

Static batching needs a full batch before it runs — bad for interactive traffic. Dynamic (or continuous) batching collects whatever requests arrived in a short window and fires them together.

```python
# Simplified dynamic batcher
class DynamicBatcher:
    def __init__(self, max_batch=16, max_wait_ms=10):
        self.max_batch = max_batch
        self.max_wait_ms = max_wait_ms
        self.queue = asyncio.Queue()

    async def _loop(self):
        while True:
            batch = [await self.queue.get()]
            deadline = time.monotonic() + self.max_wait_ms / 1000

            # Keep pulling until full OR the wait window expires.
            # We never wait *past* the deadline just to fill the batch —
            # a half-empty batch now beats a full batch 50ms later.
            while len(batch) < self.max_batch:
                timeout = deadline - time.monotonic()
                if timeout <= 0:
                    break
                try:
                    batch.append(await asyncio.wait_for(self.queue.get(), timeout))
                except asyncio.TimeoutError:
                    break

            results = await self._run_batch(batch)
            for req, res in zip(batch, results):
                req.future.set_result(res)
```

The `max_wait_ms` window is the knob that tunes the latency/throughput trade-off. 5–15ms is a sweet spot for interactive apps — imperceptible to users, but enough to catch bursts.

## Continuous Batching: The Modern Default

Dynamic batching still has a flaw: it waits for *all* sequences in a batch to finish before starting the next batch. But generations finish at wildly different lengths. One prompt answers in 10 tokens, another in 400. The short one's slot sits idle while the long one runs.

**Continuous batching** (what vLLM, TensorRT-LLM, and TGI implement) fixes this. At every generation step, it evicts finished sequences and slots new requests into the freed space. The GPU stays full.

```
Static batching:
  step: [A A A A][B B B B][C C C .]   ← C's slot wasted after it finishes
        [D D D D][E E E E][F F F .]

Continuous batching:
  step: [A B C D][E F G H][I J K L]   ← empty slots refilled every step
```

If you're building on vLLM or an inference server that supports it, this is mostly a config decision — but it's why "just use vLLM" is usually good advice over hand-rolling a server.

## KV Cache: The Memory You Keep Forgetting

Every token a model generates depends on all previous tokens. Recomputing the attention over the whole history every step is wasteful, so servers cache the keys and values per sequence — the **KV cache**.

The catch: KV cache grows linearly with context length and eats GPU memory fast. A model that "fits" for a single request can OOM with 20 concurrent long-context requests.

```python
# Rough KV cache memory, per sequence:
# bytes ≈ 2 (K and V) * layers * heads * head_dim * seq_len * dtype_bytes
#
# For a 7B model, fp16, 8k context:
# ~ 2 * 32 * 32 * 128 * 8192 * 2 ≈ 4.3 GB  per request
```

That's the real concurrency limit — not FLOPs, memory. It's why teams run quantized KV caches, paged attention (vLLM's trick), or simply cap context length.

### Quick ways to claw back memory

- **Quantize the KV cache** to int8 — roughly halves it, small quality hit.
- **Cap max context** you actually accept — don't allow 32k if your app uses 4k.
- **Limit concurrent sequences** explicitly, and queue the rest. A bounded queue with a 503 is kinder than a crash.
- **Reuse prefixes** (paged/prefix caching) when many requests share a system prompt.

## Measuring the Right Latency

"Latency" is too vague to optimize. Break it into pieces:

| Metric | What it means | Why it matters |
|--------|--------------|----------------|
| TTFT | Time to first token | Perceived responsiveness (streaming) |
| TPOT / ITL | Time per output token | Reading smoothness |
| E2E latency | Full response time | Batch jobs, tool calls |
| Throughput | Tokens/sec across all requests | Cost efficiency |

For streaming chat, **TTFT dominates perceived speed** — the user sees the first words. A high TPOT feels slow and draggy even if total time is the same. Optimize them separately; one prompt-processing tweak helps TTFT, scheduling tweaks help TPOT.

❌ Reporting a single "average latency" hides the pain.

✅ Track p50, p95, and p99 separately — the tail is what users complain about.

## A Realistic Serving Stack

```
Client ──► API gateway ──► request queue (bounded) ──► inference server ──► GPU pool
              │                    │                          │
          auth, rate           backpressure              continuous batching
          limits               on overflow               KV cache management
                                                          autoscaling signal
```

The queue is not optional. Without a bounded queue, a traffic spike turns into OOM instead of backpressure. Rejecting with 429/503 early is how you keep the requests you *can* serve fast.

```python
# ❌ Unbounded queue — a spike becomes an outage
await server.queue.put(request)

# ✅ Bounded queue — shed load before the GPU dies
try:
    server.queue.put_nowait(request)
except asyncio.QueueFull:
    raise HTTPException(503, "Server busy, retry with backoff")
```

## GPUs: What Actually Costs You

- **Utilization matters more than raw FLOPS.** A 90%-idle A100 loses to a busy, well-batched smaller GPU.
- **Cold starts are brutal.** Loading a 7B model takes seconds to minutes. Keep a warm replica or you'll pay it on every scale-up.
- **Scale-to-zero is a trap for interactive apps.** It's fine for batch; for chat it means the first user after quiet periods eats a multi-second penalty.
- **Multi-model hosts** need memory headroom math, or one model evicts another mid-request.

If you're not on your own hardware, serverless GPU/API providers hide batching but bill per token — great to start, watch the unit economics at scale.

## Trade-offs Worth Naming

- **Bigger batches** help throughput but hurt tail latency; keep the wait window small for interactive paths.
- **Longer contexts** improve quality but shrink concurrency; cap what you don't need.
- **Quantization** saves memory and cost at some accuracy cost; measure on *your* eval set, not vibes.
- **More replicas** fix latency but multiply cost and cold-start churn; autoscale on queue depth, not just CPU.

## Practical Takeaways

- Batch your requests — it's the single biggest lever on GPU cost. Use continuous batching if you can.
- Set a short dynamic batching window (5–15ms) for interactive endpoints; go big only for batch jobs.
- Budget for the **KV cache**, not just the weights. That's usually your real concurrency ceiling.
- Track TTFT and TPOT separately from end-to-end latency, and watch p95/p99, not averages.
- Put a bounded queue in front of the GPU and return 503/429 on overflow — backpressure beats crashes.
- Keep at least one warm replica for latency-sensitive traffic; scale-to-zero only for batch work.
- Don't hand-roll a server unless you have to — vLLM/TGI/TensorRT-LLM have already solved paged attention and continuous batching.
