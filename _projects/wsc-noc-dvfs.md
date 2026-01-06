---
hasThumbnail: false

---

## Background

This project was done as part of a group with Evan Li, and Nathan Sunderland for the [Architectures for Warehouse-Scale Computing class](https://ucb-wsc-fa25.github.io/). Our project explores the idea that when a server is under a power cap, not all the on-chip netwok traffic should be slowed equally. Some packets (control) are part of traffic that directly impacts responsiveness (tail latency). Other packets are "batch" traffic that can tolerate delay. We built and evaluted a priority-aware DVFS framework for a Network-On-Chip (NoC) that reallocates limited power to protect latency-critical traffic.

**Final Paper**: [Priority Aware NoC DVFS under Power Caps](/assets/pdfs/Priority_Aware_NoC_DVFS_under_Site_Power_Caps.pdf). 

Warehouse-scale computers (WSCs) often operate under site-level power caps to manage energy costs and infrastructure limits. During cap events, operators throttle lower-priority work to preserve user-facing responsiveness. Inside a server CPU, the NoC becomes a major contributor to power/performance as core counts rise, so it’s an important target for power management. We explore strategies that enable power-constrained NoCs to provide performance efficiency for traffic classes without violating global power caps.

Uniform throttling applies the same frequency/voltage reduction to all routers. It’s effective for power, but can needlessly degrade latency-critical traffic, because it slows every packet the same way.

We split on-chip traffic into two levels:
- Control-class traffic: latency-sensitive messages (e.g., coherence commands/acks, synchronization)
- Batch-class traffic: throughput-oriented transfers that can tolerate additional delay

Instead of slowing the whole fabric, we want DVFS that can throttle where it hurts least, freeing headroom to keep control paths fast. 

## Setup

We utilize a trace-driven simulation pipeline so that we can iterate quickly on controllers. The project separates trace generation from network simulation. We generate NoC traces once, and then reply them many times while changing DVFS policies.

- Sniper generates multicore NoC packet traces.
- BookSim 2.0 replays traces in a cycle-accurate NoC model while applying DVFS policies and recording power + latency metrics.

We used Tailbench++ for control-class behavior (e.g., Xapian/search, Sphinx/speech) and PARSEC for batch pressure (e.g., VIPS, Dedup).

We extended BookSim support DVFS in multi-domain up to per-router granularity. Each router maintains a continuous freq_scale that effectively changes router/link service rate. Lowering it increases queueing and tail latency under load. 

Controllers run at fixed DVFS epochs. Per epoch, we collect telemetry (power, occupancy, injection/stall rates, and per-class latency percentiles) and compute the next frequency settings within min/max bounds. To evaluate power caps realistically, we integrate the ORION 3.0 power model into BookSim’s router implementation and computes activity-driven per-router power under the current freq_scale. 

## Controllers

1) Baseline: Uniform throttling

- A conventional baseline that applies a single global frequency scale to meet the cap. It is intentionally not class-aware and cannot protect control tail latency.

1) HW Reactive

- A lightweight, hardware friendly controller that reacts to a congestion signal (queue occupancy) using hysteresis thresholds. If control P99 approaches or violates target, we force high frequency to preserve slack.
- We weight busy routers higher, and allocate them more frequency

2) Queue PID

- A per-router PID controller that regulates input-buffer occupancy toward a target and then normalizes frequencies if total power exceeds the cap.
- It can incorporate class awareness by biasing upward when control P99 is threatened.

3) Performance Target

- A controller that closes the loop with a set performance goal (control-class P99 latency). We adjust frequency based on P99 error relative to a set point.

## Results

Class-aware DVFS reduces control-class P99 latency under power caps compared to uniform throttling while staying within the budget. 

For more specific results and plots you can check out the paper attached at top and bottom of this page.

## Repositories

**Main project**: [https://github.com/YashCK/wsc-noc-dvfs](https://github.com/YashCK/wsc-noc-dvfs)

**BookSim fork**: [https://github.com/YashCK/wsc-dvfs-booksim2/](https://github.com/YashCK/wsc-dvfs-booksim2/)

**Final Paper**: [Priority Aware NoC DVFS under Power Caps](../assets/pdfs/Priority_Aware_NoC_DVFS_under_Site_Power_Caps.pdf). 
