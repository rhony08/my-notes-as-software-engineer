# Cloud Cost Optimization

Your cloud bill arrives every month. First time you see it, you do a double-take. "$3,500 for what?" You start digging and find 47 unused load balancers, a "small experiment" RDS instance that's been running for 8 months, and GPUs that nobody remembers provisioning.

I've been there. The cloud makes it _very_ easy to spend money. Here's how to stop the bleed without sacrificing reliability.

---

## Where the Money Goes

Every cloud provider follows roughly the same cost breakdown:

| Category | Typical % of Bill | The Trap |
|----------|------------------|----------|
| Compute (EC2, VMs, pods) | 40-60% | Oversized instances, idle resources |
| Storage (EBS, S3, volumes) | 15-25% | Orphaned volumes, unused snapshots |
| Networking (data transfer) | 10-20% | Cross-region traffic, NAT gateways |
| Managed services (RDS, ES, etc.) | 10-20% | Unused instances, wrong tier |

The biggest culprit? **Orphaned resources** — things someone spun up, forgot about, and are still running.

---

## Right-Sizing: The Easiest Win

Most teams over-provision. Like, _wildly_ over-provision. A common pattern: "Let's be safe, use the large instance." Then the large instance sits at 8% CPU for 6 months.

**The fix:** Monitor actual usage, then resize.

```
# Before: m5.xlarge (4 vCPU, 16 GB RAM)
# Avg CPU: 12%, Avg Memory: 3.2 GB

# After: m5.large (2 vCPU, 8 GB RAM)
# Monthly saving: ~50%
# Headroom still available for spikes
```

Cloud providers have right-sizing tools built in:
- **AWS Compute Optimizer** — recommends instance types based on historical usage
- **Azure Advisor** — same thing for Azure
- **GCP Recommender** — ditto for GCP

**Don't** resize in production without checking peak usage. Look at 90th or 95th percentile, not average. Averages hide spikes.

---

## Auto-Scaling Isn't Just for Reliability

If you're running fixed capacity 24/7, you're paying for nighttime and weekend traffic that doesn't exist.

❌ **Fixed cluster:** 10 instances × 24 hours = 240 instance-hours/day
✅ **Auto-scaled cluster:** 10 instances peak / 3 instances off-peak = ~120 instance-hours/day

That's a **50% saving** for one configuration change.

But auto-scaling isn't magic. You need:
- **Proper scale-down cooldowns** — flapping (scale up → scale down → scale up) costs _more_ than static
- **Predictive scaling** for known patterns (higher traffic at 9 AM, lower at midnight)
- **Scheduled scaling** for predictable events (black Friday, product launches)

### Spot/Preemptible Instances

For stateless workloads, spot instances are 60-90% cheaper than on-demand. Batch processing, CI runners, data pipelines — perfect candidates.

```
# Example: CI/CD pipeline using spot instances
# On-demand cost: $0.096/hour × 8 hours/day = $0.77/day
# Spot cost:      $0.029/hour × 8 hours/day = $0.23/day
# Saving: 70%
```

The catch: they can be terminated with 2 minutes notice. Not great for databases. Great for anything that can restart.

---

## Storage: The Silent Accumulator

Storage costs creep up. You don't notice the $0.023/GB-month for EBS, but 500 GB of unattached volumes? That's $138/year for _nothing_.

### Quick wins:

1. **Delete unattached volumes** — AWS has "orphaned volume" reports
2. **Clean up old snapshots** — do you _really_ need daily snapshots from 2019?
3. **Use lifecycle policies** — move old S3 objects to Glacier automatically

```
S3 Lifecycle Policy:
- Current: Standard (frequently accessed)
- 30 days: Infrequent Access
- 90 days: Glacier (archive)
- 365 days: Delete
```

4. **Right-size EBS volumes** — provisioned IOPS you're not using? Drop to gp3.

---

## Data Transfer: The Hidden Tax

Cloud providers charge for data leaving their network. It's not on the price calculator splash page, but it shows up on the bill.

- **Same AZ:** Free
- **Same region, different AZ:** Small fee
- **Cross-region:** Moderate fee
- **Internet egress:** $$$$

The biggest surprises come from:
- NAT gateways ($0.045/hour + data processing fee)
- Cross-region replication you forgot about
- Large public datasets served from expensive storage types

**Strategies:**
- Use CloudFront / CDN for public content (egress is cheaper)
- Keep services in the same AZ when latency isn't critical
- Use VPC endpoints instead of NAT gateways for AWS services
- Prefer same-region architecture

---

## Reserved Instances and Savings Plans

If you know you'll run certain workloads for 1-3 years, you can save 30-70% by committing upfront.

| Plan | Savings | Flexibility |
|------|---------|-------------|
| On-Demand | 0% | Maximum |
| 1-year partial upfront | 30-40% | Medium |
| 3-year all upfront | 60-70% | Low |
| Compute Savings Plan | ~40% | High (applies across instances) |

**Savings Plans** are generally better than Reserved Instances for most teams — they apply to any compute in a region, not a specific instance type. Less risk of "I committed to m5.large but now we're migrating to Graviton."

---

## Tagging and Accountability

You can't optimize what you can't track. Every resource should have tags:

```
Environment: production / staging / development
Team: platform / backend / data
Project: my-notes / customer-portal
Cost-Center: eng-1234
```

Then you can answer questions like:
- "How much does the staging environment cost?"
- "What's the data team spending on compute?"
- "Which customer accounts are most expensive?"

Without tagging, your cloud bill is a black box of "EC2 $4,200." That's useless for making decisions.

---

## Practical Audit Checklist

Run through this quarterly:

- [ ] **Orphaned resources** — ELBs, EBS volumes, Elastic IPs, NAT gateways
- [ ] **Idle resources** — instances with <5% CPU for 30+ days
- [ ] **Old snapshots** — AMIs and EBS snapshots older than 90 days
- [ ] **Unused reserved instances** — paying for RIs on terminated instances
- [ ] **Stale load balancers** — ALBs/NLBs with no healthy targets
- [ ] **Over-provisioned databases** — RDS instances at <20% utilization
- [ ] **Cross-region data transfer** — unexpected traffic between regions
- [ ] **Expired resources** — test environments that were supposed to live 2 days

---

## The 80/20 of Cloud Cost Optimization

If you only do three things:

1. **Right-size your compute.** Start with the top 10 most expensive instances. Chances are 3-4 are wildly over-provisioned.
2. **Auto-scale your workloads.** Fixed capacity is almost never optimal.
3. **Tag everything and generate monthly reports.** Visibility drives action.

The rest — Savings Plans, spot instances, storage lifecycle policies — is optimization _after_ you've stopped the obvious waste.

---

## Takeaways

- Cloud cost optimization is 80% about stopping waste and 20% about using cheaper pricing models
- Unattached volumes and idle instances are the easiest money to recover
- Auto-scaling saves money _and_ improves reliability — no trade-off here
- Tag everything for accountability. If you can't trace a cost to a team, it'll never be optimized
- Data transfer costs will surprise you if you don't monitor them
- Savings Plans > Reserved Instances for most teams
- Run a cleanup audit quarterly. Cloud resources have a habit of accumulating
