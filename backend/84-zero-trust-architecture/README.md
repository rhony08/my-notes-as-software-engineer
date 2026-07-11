# Zero Trust Architecture

"Trust but verify" sounds reasonable. Until you realize that trust is exactly what attackers exploit.

Here's the uncomfortable truth: the traditional perimeter security model assumes everything inside the corporate network is safe. Once you're past the firewall, you're trusted. But that assumption has been burning organizations for years—because breaches don't always come from the outside. They come from compromised credentials, malicious insiders, or attackers who *already* found a way in.

Zero Trust flips the model: **never trust, always verify.**

## What's Wrong with Castle-and-Moat?

The old security model is exactly what it sounds like: build a big wall (firewall, VPN, network segmentation) and trust everything inside. It worked when everyone worked in an office and apps ran on servers in a closet. But look at how we build things today:

- SaaS apps live outside the network
- Developers push code from coffee shops
- Services talk to each other across clouds
- Employees use personal devices

The perimeter doesn't exist anymore. You can't defend a line that's dissolved.

## What Zero Trust Actually Means

Zero Trust isn't a product you buy. It's a set of principles:

1. **No implicit trust** — authentication doesn't expire at the network boundary
2. **Least privilege** — give the minimum access needed, nothing more
3. **Assume breach** — design as if you're already compromised
4. **Never trust, always verify** — every request, every time

Think of it like an apartment building vs a house. A house has one front door with a lock (perimeter). An apartment building has a front door, a lobby, an elevator key, and a lock on every unit. Even if someone sneaks past the front door, they can't get into your apartment.

## The Core Components

### Microsegmentation

Instead of one big network segment, you break it into tiny pieces. Each piece has its own security controls.

```
// ❌ Traditional network
[Internet] → [Firewall] → [Internal Network]
                              ├── Web Server
                              ├── Database
                              └── Admin Console
                          (Everything can talk to everything)

// ✅ Zero Trust network
[Internet] → [Firewall] → [Web Segment]     → Web Server
                          [Database Segment] → Database (only accepts from Web)
                          [Admin Segment]    → Admin Console (requires MFA)
```

Without segmentation: compromising the web server means the database is wide open. With segmentation: even if the web server is compromised, the database refuses connections from it because... verification failed.

### Identity-Aware Access

Authentication is the new firewall. Every request—whether it comes from a laptop in the office or a Lambda function in AWS—must prove who (or what) it is.

This means:

- **Service-to-service auth** — mTLS, SPIFFE, or workload identity tokens
- **Device posture checks** — is the device patched? Encrypted? Running AV?
- **Context-aware policies** — "Access only during work hours, from managed devices, via approved locations"

### Continuous Verification

Here's the part most people miss: Zero Trust isn't a handshake. It's a heartbeat.

Traditional auth checks once at login, then trusts the session. Zero Trust checks **on every request**. If a user's device suddenly reports as jailbroken mid-session, access is revoked immediately—not at the next login.

## Google's BeyondCorp: A Real Example

Google ran one of the most famous Zero Trust implementations. Before BeyondCorp, Google used a VPN-based model. Employees had to VPN into the corporate network to access internal apps. It was slow, painful, and still vulnerable—once inside the VPN, you had access to everything.

BeyondCorp flipped it:

- No VPN required
- Access decisions based on user identity + device inventory + context
- Apps are accessible from the internet directly, but only after verification
- The network is treated as hostile (yes, even the internal one)

The result: employees work from anywhere without a VPN, and compromised devices get blocked without affecting anyone else.

## The Problem with Zero Trust

I wouldn't be honest if I didn't mention the downsides.

### Complexity Goes Up

Zero Trust means more moving parts:
- Identity provider (IdP) for every service
- Certificate management for mTLS
- Policy engines for dynamic decisions
- Monitoring for continuous verification

You're trading network simplicity for security granularity. That's a real operational cost.

### Latency Hits

Every request needs auth checks. Every service needs to verify credentials. Add mTLS handshake overhead on top. The latency adds up—especially in internal service mesh traffic where you previously had zero auth overhead.

### Legacy Systems Don't Play Nice

Your 10-year-old monolith probably doesn't support SPIFFE workload identities or mTLS. You'll need proxies (like Envoy sidecars) to wrap legacy services, which adds another layer to manage.

## Practical Steps to Get Started

If you're looking at this and thinking "that's a lot of work"—you're right. Nobody does Zero Trust overnight. Here's where you start:

| Step | What | Impact |
|------|------|--------|
| 1 | **Map your traffic** — understand who talks to whom | Discovery |
| 2 | **Add auth to service boundaries** — API gateways, ingress controllers | High |
| 3 | **Implement least privilege** — audit and trim IAM roles | Medium |
| 4 | **Segment critical assets** — databases, secrets, admin consoles | High |
| 5 | **Enable mTLS for internal services** — service mesh or sidecar proxies | Medium |
| 6 | **Continuous monitoring** — detect anomalies in access patterns | Ongoing |

Don't try to do all of them. Pick the highest-impact step for *your* biggest risk. For most teams, that's step 2 or 4.

## What This Means for Your Architecture

Zero Trust isn't just a security trend. It's a practical response to how systems actually work today. Your APIs are consumed by mobile apps, third-party services, internal microservices, and batch jobs. Treating them all the same way—verify everything, trust nothing—isn't paranoia. It's realistic.

The architecture implications:
- **API gateways** become policy enforcement points
- **Service meshes** (like Istio or Linkerd) become the backbone for mTLS and auth
- **Identity** becomes a first-class architectural concern, not an afterthought
- **Audit logs** become critical for incident response, not compliance checkbox

## Key Takeaways

- **Trust is a vulnerability** — the perimeter model assumes attackers can't get in, which is wrong
- **Microsegmentation limits blast radius** — a compromised web server shouldn't mean a compromised database
- **Continuous verification beats session-based auth** — check every request, not just login
- **Start small, not all-or-nothing** — secure your most critical paths first, expand from there
- **Expect higher complexity** — Zero Trust is more work, but the alternative is worse

The old model asked "are you in the network?" Zero Trust asks "should you be doing this, right now, from that device?" It's a harder question to answer—but a much better one for security.
