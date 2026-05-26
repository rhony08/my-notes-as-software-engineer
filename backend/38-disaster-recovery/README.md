# Disaster Recovery Planning

You've never had a production outage. Your database has never corrupted. A bad deployment has never wiped a table.

Okay, maybe it hasn't happened *yet*. But running a service without a disaster recovery plan is like driving without a spare tire — you're fine until you're not, and then you're really, really stuck.

Disaster recovery (DR) isn't about *if* something goes wrong. It's about answering one question: **when everything breaks, how fast can we get back up?**

---

## The Metrics That Matter

Before we talk about strategies, you need to know two acronyms that'll come up in every DR conversation:

| Metric | What It Means | Real Talk |
|--------|---------------|-----------|
| **RTO** (Recovery Time Objective) | How long until service is restored? | "My boss says we can be down for 4 hours max" |
| **RPO** (Recovery Point Objective) | How much data can we lose? | "If we lose the last 15 minutes of data, that's fine" |

These aren't technical constraints — they're **business decisions** that drive everything else.

### Putting RTO and RPO Together

```
    RPO                RTO
<--------->|<---------------->
    |                    |
Disaster happens    Service restored
    |
Data loss boundary (RPO)
```

If your RPO is 1 hour, you're comfortable losing up to an hour of data. Any backup strategy that leaves you with data older than 1 hour? Useless for this scenario.

If your RTO is 4 hours, you need a recovery plan that gets you operational within 4 hours. A multi-region failover that takes 8 hours to spin up? Doesn't meet the requirement.

---

## DR Strategies: Pick Your Trade-off

There's no "best" DR strategy — there's only the right strategy for your budget and tolerance.

### Backup & Restore (Cheapest)

```
DB snapshot every 6h ───────────→ S3/Blob Storage
                                            │
When disaster hits ─────────────────────────┘
                                            ▼
                                    Restore from snapshot
                                    + replay WAL (if possible)
```

**Pros:** Cheap. Simple. Works for most small-to-medium apps.
**Cons:** Highest recovery time. You're restoring from scratch.

**When to use:** Side projects, dev environments, low-traffic services.

### Pilot Light (Moderate)

You keep a minimal version of your infra running in a secondary region — small instance, core services only. When disaster strikes, you scale it up.

```
Primary Region (full)        DR Region (minimal)
┌────────────────────┐      ┌────────────────────┐
│ App servers x20    │      │ App server x1      │
│ DB (primary)       │─────▶│ DB (replica)       │
│ Cache cluster      │      │                    │
│ Queues + workers   │      │ Queues (paused)    │
└────────────────────┘      └────────────────────┘
                                    │
                        When DR activates:
                        Scale up app servers
                        Promote DB replica
                        Resume queue workers
```

**Pros:** Faster recovery than backup-restore. Reasonable cost.
**Cons:** Cold caches = degraded performance initially.

**When to use:** Most production SaaS apps that can't afford full hot standby.

### Warm Standby (Expensive)

Running a scaled-down but functional copy of your production environment in another region. It's already serving traffic (or ready to).

```
Primary Region            DR Region (warm)
┌────────────────┐      ┌────────────────┐
│ App servers    │      │ App servers    │ ← already running
│ DB (primary)   │─────▶│ DB (replica)   │ ← near real-time sync
│ Cache          │      │ Cache (warm)   │
│ DNS: primary   │      │ DNS: standby   │
└────────────────┘      └────────────────┘
                              │
                    Switch DNS → DR region
                    App servers take traffic immediately
```

**Pros:** Fast recovery (minutes). Warm caches. Smooth failover.
**Cons:** ~50-60% of primary region cost. Complex setup.

**When to use:** Critical services with strict SLAs (finance, healthcare).

### Multi-Region Active-Active (Most Expensive)

Both regions serve traffic simultaneously. If one goes down, the other absorbs the load.

```
Region A (EU)             Region B (US)
┌────────────────┐      ┌────────────────┐
│ App servers    │      │ App servers    │
│ DB (read-write)│←────▶│ DB (read-write)│ ← conflict resolution
│ Cache          │      │ Cache          │
│ 50% traffic    │      │ 50% traffic    │
└────────────────┘      └────────────────┘
```

**Pros:** Near-zero RTO. No traffic shifting needed. Best UX.
**Cons:** Crazy expensive. Need conflict resolution for writes. Latency across regions.

**When to use:** Global-scale apps (Netflix, Google, AWS). If you're asking "do we need this?", you probably don't.

---

## Building Your DR Plan

### Step 1: Classify Your Systems

Not everything needs gold-plated recovery. Your payment service? Yes. A slow batch-reporting job? Probably not.

```
Tier   | Example               | Target RTO | Target RPO
-------|------------------------|------------|-----------
Tier 1 | Payment processing     | < 5 min    | Near zero
Tier 2 | User profiles          | < 30 min   | < 5 min
Tier 3 | Analytics dashboard    | < 4 hours  | < 1 hour
Tier 4 | Archived reports       | < 24 hours | < 24 hours
```

Be honest with this. Tier 1 everywhere means your budget explodes for no reason.

### Step 2: Document Runbooks

When pager duty goes off at 3 AM, you won't be thinking clearly. Your runbook should be so detailed that a sleep-deprived human can execute it step-by-step.

**What a good runbook looks like:**

```
SERVICE OUTAGE: Primary DB unreachable
────────────────────────────────────
1. SSH into bastion: `ssh bastion-prod`
2. Check DB health: `pg_isready -h db-primary -U postgres`
3. If unreachable, promote replica:
   └─ `ssh db-replica-1`
   └─ `pg_ctl promote` (PostgreSQL) or failover via orchestrator
4. Update DNS to point to new primary
5. Verify app can connect:
   └─ `curl -X GET https://api.internal/health/db`
6. Notify on-call channel: @channel DB failover completed
```

❌ **Bad runbook:** "Fail over the database" (too vague, too many ways to mess up)

✅ **Good runbook:** Exact commands + expected output + rollback instructions

### Step 3: Automate What You Can

Manual steps are where disasters compound. A junior engineer hitting the wrong button can turn a 15-minute recovery into hours of pain.

```
❌ Manual failover:
   1. Someone SSH-es into the server
   2. Types commands from memory
   3. Hopes it works

✅ Automated failover:
   1. Monitored health check fails
   2. Orchestrator detects and promotes replica
   3. DNS record automatically updates
   4. On-call gets notification: "Failover completed"
```

**Low-hanging fruit:**
- Database auto-failover (RDS Multi-AZ, PostgreSQL repmgr)
- DNS health checks with automatic routing (Route 53, Cloudflare)
- Infrastructure as code — so you can rebuild from scratch
- Automated backup verification (restore snapshots weekly)

### Step 4: Test. Then Test Again.

A DR plan you've never tested isn't a plan — it's a wish.

**Testing approaches:**

| Level | What You Do | Frequency | Found Issues |
|-------|-------------|-----------|--------------|
| Tabletop | Walk through the scenario as a team | Monthly | Process gaps, unclear ownership |
| Partial | Fail over one non-critical service | Quarterly | Credential issues, DNS problems |
| Full | Simulate region failure, run DR | Bi-annual | Everything: capacity, timing, data loss |
| Game Day | Random surprise test | Annual | How the team actually reacts under pressure |

**Real talk:** Every company I know that runs Game Days finds something broken. Every single time. That's the point — find it during testing, not during a real outage.

---

## Common Pitfalls (Learned the Hard Way)

### "We'll just restore from backup"

```
Scenario: Someone accidentally DROPPED a table.
Backup from 6 hours ago: ✅ Available
RPO: 6 hours of lost data → ❌ Business says unacceptable
Time to restore 5TB: 4 hours → ❌ RTO violated
```

Backups are necessary but not sufficient. You need to know your actual recovery time and data loss.

### DNS Propagation Delays

You switch your DNS record to the DR region. Great. Now you wait 5 minutes… 30 minutes… 2 hours… because some DNS resolvers cached the old record.

**Fix:** Use low TTL on critical records (60 seconds) during an incident, or use DNS health check systems that update instantly.

### Untested Credentials

The DR runbook says "use the backup vault in DR region." Except no one set up IAM roles in that region. Or the encryption keys are in a different KMS. Or the database password is hardcoded and only works in primary.

**Fix:** Your DR infrastructure must be identical in setup. Use IaC. Test credentials as part of every DR drill.

### Overlooking Stateful Services

You've got the database covered. But what about:
- Message queues with unprocessed events
- Redis cache with session data
- File uploads stored on local disk (not S3)
- Scheduled cron jobs that run once

Every piece of state needs a DR story. If your queue has 50,000 unprocessed messages and the primary goes down — where do those messages go?

---

## Recovery Runbook Checklist

When disaster strikes, work through this:

- [ ] **1. Assess** — Is it a blip or a real disaster? (5 min)
- [ ] **2. Declare** — Official DR activation. Everyone knows.
- [ ] **3. Failover** — Execute your automated/manual failover
- [ ] **4. Verify** — Health checks pass? Traffic routing correctly?
- [ ] **5. Communicate** — Status page, internal chat, stakeholders
- [ ] **6. Mitigate** — Can primary be salvaged? Or is DR permanent?
- [ ] **7. Post-mortem** — What went wrong? How to prevent recurrence?

---

## Takeaways

- **RTO and RPO are business decisions, not tech ones.** Talk to your stakeholders before picking numbers.
- **Backup & restore is the baseline.** Anyone running a production service should have at least this.
- **Automate everything you can.** Manual failover adds stress and error at the worst possible time.
- **Test until it hurts.** If your last DR test was "six months ago and it passed," you're overdue.
- **Stateful services are the hardest part.** DB, queues, caches, file storage — every one needs a plan.
- **Document runbooks like you're explaining to a sleep-deprived colleague.** Because you probably will be.
