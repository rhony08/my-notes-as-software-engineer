# Profiling Tools and Techniques

Your service is slow. Not "we need more servers" slow — more like "requests take 2 seconds and nobody knows why" slow. You've checked the queries. You've added logging. You've even guessed and optimized a few functions you *felt* were slow. Nothing moved the needle. Sound familiar?

Here's the uncomfortable truth: in production, your intuition about what's slow is usually wrong. I've "optimized" plenty of code that turned out to be 2% of the problem. The only reliable way to find real bottlenecks is to measure — and that's exactly what profiling is for. This article covers the profiling tools worth knowing (pprof, perf, py-spy, async-profiler), how to read flame graphs, and a workflow that doesn't waste your afternoon.

## Why Guessing Fails

Our brains are terrible at estimating where time goes in code. A function that runs once but takes 10ms feels slow. A function that runs 50,000 times at 0.1ms each is invisible in the code — but it's 5 seconds of your total time. You cannot spot that by reading.

Profiling flips this: instead of asking "what *could* be slow?", it measures "what *is* slow?" across actual runs. The results regularly surprise people. The classic example — I once found a "slow database query" that was actually 90% JSON serialization of a huge response. The query was fine. The serializer was the problem.

## Sampling vs Instrumentation

Before tools, the one concept you need: how profilers actually collect data.

**Sampling profilers** (most common) periodically interrupt the program — every 1-10ms — and record where it's executing. They build a statistical picture: "we sampled 500 times, 300 of those were in `serialize()`". They're cheap (1-3% overhead), safe for production, and *approximate*. Miss a rare code path and it just won't show up.

**Instrumenting profilers** wrap every function call with timing code. Exact measurements, call counts, per-call averages — but 10-100x overhead, and they distort the thing you're measuring (the instrumentation itself takes time). Fine for a staging environment, risky in production.

Rule of thumb: **sample in production, instrument when you need exact counts.** For 95% of "why is this slow" questions, sampling is enough.

## The Tools, by Ecosystem

The good news: every major ecosystem has a solid profiler, and they all output something you can visualize the same way.

| Tool | Ecosystem | What it does best | Overhead |
|------|-----------|-------------------|----------|
| `pprof` | Go | CPU, heap, goroutine, mutex profiles; built-in HTTP endpoint | Low |
| `async-profiler` | Java/JVM | CPU + allocation + wall-clock; flame graph output | Low |
| `py-spy` | Python | Samples a *running* process, no code changes, no restart | Very low |
| `perf` | Any native/Linux | CPU + hardware counters (cache misses, branch misses) | Very low |
| `clinic.js` | Node.js | CPU, memory, event-loop delay — nice visual output | Medium |
| `valgrind`/`callgrind` | Native (C/C++) | Exact instruction counts, cache simulation | Huge (10-50x) |

## A Concrete Walkthrough: Go's pprof

Go ships the best profiler-to-flamegraph story of any language, so it's worth seeing end to end. You enable it with two lines:

```go
import _ "net/http/pprof"  // registers /debug/pprof/ endpoints on the default mux

// in main():
go func() {
    log.Println(http.ListenAndServe("localhost:6060", nil))
}()
```

That's it. Now you have live endpoints. The ones you'll actually use:

- `/debug/pprof/profile?seconds=30` — CPU profile (the big one)
- `/debug/pprof/heap` — heap allocations
- `/debug/pprof/goroutine` — goroutine dump (deadlock hunting)
- `/debug/pprof/mutex` — contended mutexes

Grab a CPU profile while your service is under load, then look at it:

```bash
# Capture 30s of CPU data while traffic is flowing
curl -o cpu.pprof "http://localhost:6060/debug/pprof/profile?seconds=30"

# Interactive REPL: type "top" to see hottest functions
go tool pprof cpu.pprof

# Or skip straight to a flame graph in your browser
go tool pprof -http=:8080 cpu.pprof
```

The `top` view is where you start — it's the profile equivalent of a leaderboard:

```text
flat  flat%   sum%        cum   cum%
120ms  5.97% 12.39%     310ms 15.42%  encoding/json.(*Encoder).Encode
 90ms  4.48%  9.45%      90ms  4.48%  runtime.memmove
 80ms  3.98%  8.46%     580ms 28.86%  myapp/internal/orders.(*Service).List
```

Read it as: `flat` is time spent *inside* the function itself, `cum` includes everything it called. `List` looks expensive (580ms cumulative) but it's mostly calling into other things. `json.Encode` is 310ms of *its own* work — that's a real target.

## Flame Graphs: Reading the Heat

Every profiler worth using can produce a flame graph (Brendan Gregg made them famous, and they're the best way to *see* a profile). They look intimidating, but the rules are simple:

- **Bottom-up:** the x-axis is time, the y-axis is call depth. The bottom row is what's actually running.
- **Width = time.** A wide block means lots of time there. That's your bottleneck.
- **Colors are meaningless** (usually just alternating shades to separate frames).
- **Look for wide flat tops** — a wide block with nothing above it means the function itself is the cost, not its children.

What you're hunting: a wide, flat frame near the top of the stack = self time = a function that's slow *by itself* (bad algorithm, allocations, syscalls). A wide frame with lots of children stacked above it = the *caller* is fine, something it calls is the problem — click down into it.

![A flame graph showing a wide JSON encoding frame](https://brendangregg.com/FlameGraphs/cpu-mysql-io.svg)

> The SVG above is an example — the shape matters more than the colors. Wide frame at the bottom with nothing above it: that's the function eating your CPU.

## The Wall-Clock Trap: CPU Isn't Everything

Here's the mistake that sends people in circles: CPU profiles only show *CPU* time. If your problem is waiting — on I/O, locks, network, the database — a CPU profile will show your process as mostly idle and tell you nothing useful.

Three situations where CPU profiling lies to you:

1. **I/O-bound work.** Your service spends 800ms of every request waiting on a downstream API. CPU profile says "99% idle." Of course — you're not *computing*, you're *waiting*.
2. **Blocked on locks/mutexes.** Thread A holds a lock, everyone else waits. CPU profile shows idle threads; the `mutex`/`block` profile shows the contention.
3. **GC / memory pressure.** The time isn't in your functions at all — it's the garbage collector running constantly because you're allocating like crazy.

For each case, use the right profile:

| Symptom | Wrong tool | Right tool |
|---------|-----------|------------|
| "Requests are slow but CPU is low" | CPU profile | Wall-clock profile, tracing, `perf` on off-CPU |
| "Everything is stuck / high latency spikes" | CPU profile | Block/mutex profile, lock contention stats |
| "CPU high but no obvious hot function" | Top-down only | Heap/allocation profile — it's GC pressure |
| "Slow only under load" | Single-shot profile | Continuous profiling (see below) |

Go's pprof has `-seconds` wall-clock via `trace`; Java's async-profiler has a `wall` mode (`-e wall`) that samples regardless of CPU state. py-spy is wall-clock by default, which is why it's great for finding "waiting" problems in Python.

## Profiling a Running Python Service Without Restarting

Python's standard `cProfile` requires you to run your code through it — which means restarting, which means dev environment, which means not the real problem. `py-spy` fixes this: it attaches to a *running* process like a debugger and samples it.

```bash
# Attach to PID 4242, dump a 30s CPU profile to a flame graph file
py-spy record --pid 4242 --duration 30 -o profile.svg

# Or get a live top view, no files needed
py-spy top --pid 4242
```

No code changes. No restart. Works on processes you didn't write and can't easily modify (a colleague's service, a library doing something dumb). This is the tool I reach for when someone says "our Python service is slow in prod but fast locally" — because the fix for that is almost never more local profiling.

## The Java Story: async-profiler

Java deserves its own mention because the JVM's `-agentlib:hprof` is ancient and the modern answer is async-profiler (now bundled in JDK Flight Recorder workflows). It uses `perf` under the hood to sample native *and* JIT-compiled code without safepoint bias (older Java profilers only sampled at safe points, which distorted results badly).

```bash
# 60s CPU profile, flame graph HTML output
./profiler.sh -d 60 -f flamegraph.html <pid>

# Allocation profile — find what's hammering the GC
./profiler.sh -d 60 -e alloc -f alloc.html <pid>
```

The `-e alloc` mode is the killer feature. Java's #1 perf problem is usually allocation pressure, and this shows you exactly which allocation sites generate the most garbage. Fix those and GC pauses shrink dramatically — I've seen services cut p99 latency in half from allocation profiling alone.

## A Workflow That Doesn't Waste Your Day

You don't need to become a profiling expert. You need a repeatable process. Here's mine:

1. **Get a baseline.** Before touching anything, capture a profile. You need "before" to know if your change did anything.
2. **Reproduce under realistic load.** A profile of an idle service is useless. Generate real traffic (or replay production traffic if you can) and profile during it.
3. **Find the widest frame, then the widest *flat* frame.** The wide frame tells you the area; the flat frame tells you the actual cost inside it.
4. **Fix one thing, re-profile, compare.** If the widest frame didn't shrink, your fix did nothing. Move on.
5. **Stop at "good enough."** You're looking for the 80/20 — the one or two things that dominate. The remaining 5% cost more engineering time than it's worth.

```bash
# Minimal repeatable loop: profile before, fix, profile after, diff
# (Go example, same idea everywhere)
curl -o before.pprof "localhost:6060/debug/pprof/profile?seconds=30"
# ... fix the hot function ...
curl -o after.pprof "localhost:6060/debug/pprof/profile?seconds=30"
go tool pprof -top before.pprof | head -10   # compare manually
```

```go
// ❌ Profiling while the service is idle — you'll learn nothing
// ("top" shows runtime routines, your code doesn't even appear)

// ✅ Profile under real load, then the wide frames mean something
// generate traffic: hey -z 30s http://localhost:8080/api/orders
// then: curl -o cpu.pprof "localhost:6060/debug/pprof/profile?seconds=30"
```

## Continuous Profiling: The Production Upgrade

One-shot profiling is a snapshot. Problems like memory leaks or slow growth over time are invisible in a 30-second window. That's where continuous profiling tools come in — they sample production services 24/7 and store the profiles, so when something goes wrong you can look at *what the profile looked like an hour before the incident*.

The big names: **Google Cloud Profiler**, **Datadog Continuous Profiler**, **Pyroscope** (open source, self-hostable). The practical value: an incident happens at 2am, and instead of hoping you can reproduce it, you pull up the flame graph from 1:45am and see the memory growth directly. For any service with on-call duty, this pays for itself after one bad night.

Trade-off to know: continuous profiling costs a few percent CPU permanently, plus storage for all those profiles. For most teams, profiling the top 5-10 services (not everything) is the sweet spot.

## When Profiling Won't Help

Honest limits, because this article would be incomplete without them:

- **Obvious architectural problems first.** If your service does a synchronous HTTP call to another service *inside* a database transaction, profiling won't save you. Fix the design.
- **Micro-optimization rabbit holes.** Profiling will happily tell you that `memmove` is 4% of your time. That's usually not worth chasing. The wins are in the wide frames.
- **Distributed latency.** If the slowness spans services, a single-service profile shows one piece. You need distributed tracing (a whole other topic) to see the full path.
- **Environment drift.** The profile from your laptop with 16 cores and nothing else running won't match production. Profile where the problem is, or at least under load.

## Takeaways

- Your intuition about what's slow is unreliable — measure first, optimize second.
- Sampling profilers (pprof, async-profiler, py-spy, perf) are cheap and production-safe; instrumentation is for exact counts in staging.
- Flame graphs: width = time, wide flat frames = self-inflicted cost, wide frames with children = look deeper.
- CPU profiles are blind to I/O, locks, and GC — pick the profile type that matches your symptom.
- Always capture a "before" profile; if the wide frame doesn't shrink after your fix, the fix didn't work.
- For services with on-call, continuous profiling turns "we can't reproduce it" into "here's the flame graph from 1:45am."
