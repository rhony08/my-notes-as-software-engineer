# Chaos Engineering Basics

Your system is running fine in staging. Tests pass. Deployments are smooth. Then a single network partition in production takes down your entire service for 40 minutes, and you're scrambling to figure out what happened.

That's why chaos engineering exists — not to break things for fun, but to find weaknesses before they find you.

## What Chaos Engineering Actually Is

Let's clear something up first: chaos engineering isn't about randomly killing processes and seeing what happens. It's a **disciplined, scientific approach** to building confidence in your system's ability to handle turbulent conditions.

Think of it like fire drills. You don't set your office on fire to test the alarms. You simulate smoke, observe behavior, and fix the gaps. Same idea, but for distributed systems.

The core loop:

```
1. Define "normal" behavior (steady state)
2. Inject a real-world failure
3. Observe what breaks
4. Fix it
5. Repeat with bigger failures
```

## The Steady State Hypothesis

This is the foundation. Before you break anything, you need to know what "normal" looks like for your system.

**Pick metrics that matter:**

- p50/p95/p99 latency for critical endpoints
- Error rates
- Throughput (requests/second)
- Resource utilization (CPU, memory, connections)

Your hypothesis looks like: "If we kill one of the three API instances, the system will continue serving with p99 latency under 500ms and 0 errors."

Then you test it.

## What Kind of Failures to Inject

Real-world failures, not theoretical ones:

| Failure | What It Simulates | Typical Impact |
|---------|------------------|----------------|
| Instance termination | Spot instance reclaim, node failure | Traffic redirect to remaining instances |
| Network latency +100ms | Cross-region degradation, bad routing | Timeouts, retry storms |
| Network partition | DNS issues, firewall misconfig | Split-brain, incomplete writes |
| CPU/memory spikes | Noisy neighbors, runaway processes | Throttling, OOM kills |
| Certificate expiration | Expired TLS certs (yes, this happens) | Complete connection failures |
| Database failover | Primary DB crash | Read replicas go stale, writes fail briefly |
| Dependency slowdown | Downstream API degradation | Cascading failures, thread pool exhaustion |
| DNS failures | DNS provider issue | Connection refused on all external calls |

You don't need to simulate all of these on day one. Start with the ones most likely to hit you.

## Blast Radius: The Golden Rule

**Start small.** Your first chaos experiment shouldn't take down production for all users. That's not an experiment — that's a disaster.

```go
// ❌ Starting with full traffic kill
if shouldInjectChaos() {
    killAllInstances()  // BAD: affects all users
}

// ✅ Start with a subset of traffic
type ChaosConfig struct {
    Enabled        bool
    TrafficPercent int    // Start at 5%
    TargetRegion   string // Start with one region
    MaxDurationSec int    // Auto-rollback after 60s
}

if shouldInjectChaos() && rand.Float64() < 0.05 {
    injectLatency(TargetService, 200*time.Millisecond)
    go autoRollback(60 * time.Second)
}
```

The blast radius chain should look like:

```
One user → One service → One region → All regions → Production
```

Don't skip steps. Each is a learning opportunity.

## Observability: You Can't Chaos Without It

Here's the thing — chaos engineering only works if you can actually **see** what breaks. If your monitoring is weak, injecting failures is just creating blind chaos.

You need:

- **Distributed tracing** — to see where the failure propagates
- **Metrics** — latency, error rates, saturation (USE method)
- **Logs** — structured, searchable, correlated

If someone asks "what happened" and you can't answer within 30 seconds, your observability isn't ready for chaos experiments.

## Tools to Get Started

You don't need a complex platform to start. Here are practical options:

### Chaos Monkey (Netflix)
The OG. Terminates instances randomly in production. Forces you to build systems that survive instance loss. Great for proving your auto-scaling and failover actually work.

### Gremlin
Commercial, but has a generous free tier. Let's you inject CPU, memory, latency, and network failures with a nice UI and blast radius controls. Good for teams that want guardrails.

### Litmus (Open Source)
CNCF project. Runs experiments as Kubernetes jobs. Good if you're already on K8s and want to define experiments as YAML:

```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosExperiment
metadata:
  name: pod-delete
spec:
  definition:
    scope: Namespaced
    probe: 
      - name: "check-app-health"
        type: httpProbe
        httpProbe/inputs:
          url: "http://app-service:8080/health"
          insecure: "skip"
    experiments:
      - name: pod-delete
        spec:
          components:
            env:
              - name: TOTAL_CHAOS_DURATION
                value: "30"
              - name: RAMP_TIME
                value: "10"
              - name: FORCE
                value: "true"
```

### Chaos Mesh
Another CNCF option from PingCAP (the TiDB folks). Runs natively on Kubernetes with a web UI. Supports pod kill, network partition, DNS chaos, and stress testing.

## The GameDay Pattern

A GameDay is a scheduled, planned chaos experiment. It's not a surprise — everyone knows it's coming. The point isn't catching people off guard; it's practicing the response.

**Anatomy of a good GameDay:**

1. **Pick one hypothesis** — don't test everything at once
2. **Schedule it** — 2 weeks in advance, during business hours
3. **Prep the runbook** — what to look for, when to abort
4. **Assign roles** — one person runs the experiment, one person watches metrics, one person is on standby to abort
5. **Run it** — inject the failure, observe
6. **Retro** — what broke, what didn't, what surprised us

**Example GameDay runbook:**

```
Objective: Verify that killing 1 of 3 payment-service pods doesn't increase p99 latency above 600ms
Duration: 15 minutes
Blast radius: payment-service only, staging environment

Steps:
1. Baseline: Record current latency (p50=45ms, p99=120ms, error=0%)
2. Inject: Kill 1 payment-service pod via Chaos Mesh
3. Observe: Watch latency for 3 minutes
4. If p99 exceeds 600ms → ABORT
5. If errors > 0.1% → ABORT
6. Restore: Verify pod comes back and traffic rebalances
7. Document findings
```

## The Hard Truth: Production vs Staging

Everyone asks: "Should I run chaos experiments in production?"

The short answer: real confidence comes from production experiments. But you don't start there.

```
Staging → Canary → Production (small scope) → Production (full scope)
```

The difference between staging and production is that staging tells you what **might** break, and production tells you what **will** break when it matters.

Netflix runs Chaos Monkey in production because they've built the guardrails over years. Start in staging, build confidence, eventually experiment in production with a 1% blast radius.

## When NOT to Do Chaos Engineering

Chaos engineering isn't always the right answer:

- **You have no monitoring** — fix observability first
- **Your system is already unstable** — fix the fires before lighting new ones
- **You have no rollback plan** — every experiment needs an abort button
- **Your on-call team is burned out** — chaos experiments add cognitive load
- **Regulatory constraints** — some industries (healthcare, finance) have strict rules about production experiments

Be honest about where you are. Chaos engineering is a maturity practice, not a checkbox.

## Taking Action

Start small. Pick one thing:

1. **This week:** Document your steady state metrics (latency, error rate, throughput)
2. **Next week:** Run a GameDay in staging with a single pod kill
3. **Month 2:** Increase scope — network latency, dependency slowdown
4. **Quarter 2:** Run in production with 1% blast radius

Chaos engineering is a journey, not a one-time project. The goal isn't to break things — it's to build systems that don't break when things go wrong.
