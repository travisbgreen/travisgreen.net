---
layout: post
title: "The Port Scoping Paradox: When Optimization Makes Things Slower"
date: "2026-02-12"
tags: suricata performance optimization detection-engineering
---

**TL;DR:** Port scoping in Suricata rules — a widely recommended optimization — can actually *increase* CPU usage by 20-30% when the traffic you're analyzing is already on the target ports. Here's the story of how I discovered this counterintuitive behavior and what it means for rule development.

---

## The Setup: An "Obvious" Optimization

You're writing Suricata detection rules for RDP brute force attacks. Your starting point looks like this:

```suricata
alert tcp any any -> any any (
    msg:"RDP Brute Force Attempt";
    content:"|e0 00 00 00 00 00|Cookie|3a 20|";
    offset:5; depth:14;
    sid:7704787;
)
```

A colleague suggests what seems like an obvious win: "Why inspect every TCP connection? RDP runs on port 3389. Scope it to the port!"

```suricata
alert tcp any any -> any [3388,3389] (
    msg:"RDP Brute Force Attempt";
    content:"|e0 00 00 00 00 00|Cookie|3a 20|";
    offset:5; depth:14;
    sid:7704787;
)
```

The logic is bulletproof, right? Most traffic isn't RDP — maybe 1% of connections. Port scoping filters out 99% of packets before any expensive pattern matching happens. You save 99% of CPU on these rules.

Right?

## The Twist

We ran the numbers. Single-threaded profiling mode, 5 iterations for statistical confidence, 376,460 packets of real network traffic. Both rule variants evaluated **the exact same packets** — "Checks" counts how many packets each rule fully evaluated.

| Rule config | Checks | CPU ticks | Result |
|:---|:---|:---|:---|
| `any any` | 1,010 | 382,581 | Baseline |
| `[3388,3389]` | 1,010 | 496,132 | **+29.7% overhead** |

**🔍 Critical insight:** Same checks (1,010), but port-scoped rules burned 113,551 extra CPU cycles doing the same work.

Look at the Checks column: both configurations evaluated **exactly the same 1,010 packets**. Port scoping did not prefilter anything. Because the test traffic was already RDP on port 3389, every packet matched the port condition — so the rule still ran on all 1,010 of them. The "optimization" never got to skip a single packet. It only added overhead to each one.

The result: 113,551 extra CPU cycles for the same amount of work.

I ran it again five times to be sure:

| Iteration | Overhead |
|:---|:---|
| 1 | +26% |
| 2 | +37% |
| 3 | +32% |
| 4 | +26% |
| 5 | +29% |

Coefficient of variation: 4.9% (anything under 10% is reliable). This wasn't noise.

## Why?

To understand what went wrong, you need to know what Suricata's prefilter actually does.

### Prefilter Uses fast_pattern, Not Ports

Suricata's prefilter is built around `fast_pattern` — the content keyword that feeds the MPM (multi-pattern matching) engine. Only packets that hit a fast_pattern go on to full rule evaluation.

**Port conditions aren't part of prefilter.** They're checked *during* full rule evaluation, after MPM has already decided the packet is a candidate.

### What Port Scoping Actually Adds

When you write `any [3388,3389]`, you're not adding an early exit. You're adding an extra check that runs inside every rule evaluation:

**With `any any`:**
1. MPM scan → hit on content
2. Rule evaluation → content check → match

**With `[3388,3389]`:**
1. MPM scan → hit on content
2. Rule evaluation → **port list traversal** → content check → match

For packets on port 3389 that match the content (all 1,010 in our test), step 2 now does *more work*, not less.

### The Port Matching Overhead

With `any any`, Suricata sees `ANY_PORT` and the compiler eliminates the check entirely. With a port list, a linked list must be traversed at runtime on every check:

**Compiled away:**
```c
// any any: optimized away by compiler — zero cost
if (port_spec == ANY_PORT) {
    return true;
}
```

**Runtime traversal:**
```c
// [3388,3389]: runtime traversal on every evaluation
for (DetectPortRange *range = port_spec->head; range != NULL; range = range->next) {
    if (packet_port >= range->low && packet_port <= range->high) {
        return true;
    }
}
return false;
```

The port list structure sits in L2/L3 cache, not the hot L1 cache where rule structures live. Every evaluation pays a cache miss to fetch it. Across 1,010 checks, the estimated ~120-240 cycles per lookup produces the 113,551 extra cycles we measured.

The overhead breaks down to:

| Source | Share | Detail |
|:---|:---|:---|
| Memory access penalty | ~60% | Port list lives in L2/L3 cache; rule structures are hot in L1. Every lookup pays a ~100-200 cycle cache miss. |
| Function call overhead | ~20% | The compiler can optimize `if (ANY)` to nothing. Port list traversal is runtime data — it can't be eliminated. |
| Conditional branch overhead | ~20% | Extra branch instructions, potential mispredictions, loop bookkeeping. |

At the per-rule level for SID 7704787 the picture was even starker: 2,836 ticks per check without port scoping vs 8,420 with it — a +197% increase on a cold-cache run with only 2 checks. This is a worst-case result (no cache warmth), but it illustrates the mechanism clearly.

## The Break-Even Point

The overhead is predictable enough to model. From the aggregate results above: 382,581 total ticks / 1,010 checks ≈ **379 ticks** average baseline evaluation; (496,132 − 382,581) / 1,010 ≈ **112 ticks** added overhead per check from port matching. The break-even is:

```
Break-even = T_base / (T_base + T_port)
           = 379 / (379 + 112)
           ≈ 77%
```

Port scoping saves CPU only when fewer than **~77% of your TCP traffic** is on the target ports. Plotted out:

| Network | RDP % | Effect | Use scoping? |
|:---|:---|:---|:---|
| General internet monitoring | 0.01% | +99.9% faster | Yes |
| Enterprise network | 5% | +77% faster | Yes |
| Security lab (mixed) | 50% | +39% faster | Yes |
| RDP-focused investigation | 85% | -11% slower | No |
| Pure RDP capture | 100% | -27% slower | No |


## Practical Takeaways

**Port scoping works well when:**
- You're monitoring diverse production traffic (most common case)
- The target protocol is rare (< ~80% of your TCP traffic)
- You need to reduce false positives from rules triggering on wrong ports

**Port scoping hurts when:**
- You're replaying or analyzing targeted captures
- Your traffic is already filtered (an RDP-only PCAP, a dedicated monitoring sensor)
- Your rules are already lightweight and not consuming meaningful CPU

For RDP brute force rules specifically, I landed on keeping `any any` because:
- We already have a solid `fast_pattern` for prefilter
- The rules are already cheap (< 0.1% CPU in a 6,477-rule set)
- RDP on non-standard ports is a real and common evasion technique
- The profiling showed a 27% degradation with port scoping on **this** traffic


## Measure, Don't Assume

We assumed port scoping would save 90-95% CPU. The reality was -27%.

**Before tuning rules:**

1. **Profile first:** `suricata -r test.pcap -S rules.rules --profile`
2. **Find hot rules:** Check `rule_perf.log` — ignore rules using < 0.1% CPU
3. **Know your traffic:** What % is actually on your target port?
4. **Test both ways:** Measure with and without scoping
5. **Deploy the faster one** — not the one that "should" be faster

Port scoping isn't wrong — it's just situational. Documentation says it's best practice, and for most environments it is. But "most" isn't "all."

The difference between good detection engineering and great detection engineering is measuring your assumptions.

---

*Test environment: Suricata (containerized), single-threaded profiling mode, 376k packet PCAP of real network traffic, 5 iterations, CV 4.9%.*
