---
hasThumbnail: false

---

## Background

This project was done as part of the EECS 251B class as a final project. Half the class worked on a USB 2.0 chip while the other half worked on an Ethernet chip. I was on the Blind Data Recovery (BDR) team on the USB 2.0 chip.

In USB 2.0, the receiver does not get a forwarded clock. Only the data signal is transmitted - so the RX must recover timing from data transitions. This is especially challenging under clock drift, jitter, and long runs without transitions (consecutive identical digits), which reduce timing margin.

Traditional USB PHYs often use PLL or DLL-based CDR. These work well, but they introduce sensitive analog loops and practical drawbacks: PVT sensitivity, design complexity, and non-trivial lock time, which can be challenging for packetized/bursty links like USB.

A blind oversampling CDR samples the incoming data with a free-running clock at a fixed multiple of the bit rate and then picks the best sample point digitally. In the reference design I studied (Park et al.), this approach is attractive for USB because it is fully digital (easier to port across processes) and offers fast lock time due to its feed-forward structure.

Our class project followed this direction: a fully digital RX CDR using **5x oversampling** plus an **Add-Drop FIFO** to correct timing-mismatch artifacts.

## System overview

At a high level, the block converts an incoming 480 Mb/s serial stream into a clean recovered bitstream (and then into bytes for downstream logic), even when:
- the transmitter and receiver clocks have small frequency offset
- jitter pushes transitions toward sampling boundaries.

The implementation is a feed-forward pipeline with three core stages:
1. **Oversampler (5x at 2.4 GHz)**: collects a 5-sample window per bit  
2. **Coarse Data Recovery (CDR)**: finds transitions, chooses the best phase, emits insert/drop signals
3. **Add-Drop FIFO (AD-FIFO)**: removes duplicated bits or inserts a missing bit to re-align the stream

**Clock domains:** oversampler at **2.4 GHz**, CDR + AD-FIFO at **480 MHz**.

## Key Ideas

1) Center-Picking Intuition

With 5x oversampling, each bit period yields 5 candidate samples. The goal is to pick the sample that's closest to the center of the data eye (highest noise/timing margin). We detect where the transition happened in the sampling window, and then pick a phase offset into the stable region.

2) Add-Drop FIFO

If the TX and RX clocks are slightly different, the transition location will draft across the 5-sample window over time and can hit a boundary.

When that happens, the coarse recovered stream can contain a duplicate bit (same bit recovered twice) or a missing bit (one bit skipped). We use ADD/DROP signaling to trigger correction in a downstream AD-FIFO.

## Block-by-Block

#### Oversampler (5x)

This sub-block captures 5 time samples of the incoming serial data per period using a 2.4 GHz clock. This is done via a 5-stage DFF shift register that holdes samples [0..4]. This 5 bit window gives us some wiggle room, such that even if the ideal sampling point drifts, we often still have one sample near the eye center.

#### Coarse Data Recovery (transition detect → phase select → data sample)

The CDR’s job is to detect where an edge occurred within the 5-sample window and choose a stable sampling phase for the bit. The first part of this is an edge detector: we flag transitions by comparing to the previous window's last sample (XOR-based). The second part of this is a phase selector (center-picking mapping). 

We used a simple mapping from edge at position i to sample at position j:
- edge at 0 → select 2  
- edge at 1 → select 3  
- edge at 2 → select 4  
- edge at 3 → select 0  
- edge at 4 → select 1  

If no edge is seen, then we retain the previous phase. Since some patterns can produce ambiguous multi-edge indications. We handled cases like `11011` or `00100` with a small voting/stabilization mechanism across adjacent windows.

We then select one the 5 samples using a 5:1 mux driven by the chosen phase pointer, clocked at the bit rate domain (480 MHz), and latch the output for clean handoff.

#### Add-Drop FIFO (AD-FIFO)

Under drift/jitter, the coarse recovered stream can contain missing or duplicated bits. If we ignore this, the error can accumulate and corrupt the packet. We have a 25-entry bit-level shift register FIFO, and a token pointer which selects which FIFO cell is currently the output. Each cycle we shift in the next recovered bit. Token movement implements add/drop behavior.

**Three cases:**

1) Normal: Shift in bit, token is unchanged

2) Drop: Detected a duplicate so we shift in bit, and decrement token (effectively skips one duplicated output bit)

3) Insert: Detected a miss. We use a 2-cycle FSM to insert a dummy bit, and then resume normal shifting

We use an inverted next bit in insert because the missing bit occurs right after a transition. In that scenario, the missing bit is the inverse of the following bit, so inserting an inverted value should reconstruct the intended sequence. We also clamp the token at boundaries to avoid overflow/underflow. The depth of 25 bits is chosen as USB packets can be upto 8255 bits and must tolerate a frequency offset of ±500 ppm.

## Verification and Results

We verified the oversampler, CDR, and AD-FIFO with targeted unit tests (transition patterns, ambiguous windows, insert/drop sequences) and then ran integration tests with injected drift/jitter conditions.

We achieved
- Correct operation under **±1000 ppm** frequency drift in simulation
- **Lock time:** feed-forward recovery starts immediately and the AD-FIFO has a warm-up latency of 12 cycles
- **Area (SKY130 synthesis):** total ≈ 9,969 µm^2 (AD-FIFO dominates)

Here is a paper we wrote for our final report: [BDR Final Report](/assets/pdfs/BDR_Final_Report.pdf). 