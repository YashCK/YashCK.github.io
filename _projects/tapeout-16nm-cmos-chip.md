---
hasThumbnail: false

---

## Background

The EE 194 (tapeout) course at UC Berkeley was made possible through the help of postgraduates, Berkeley Wireless Research Center, and Apple design reviews. In the class undergraduates collectively come together to design and send chips out to fabrication. This process involves the microarchitecture and RTL design, SoC integration, verification, and physical design. In Spring 2025 the chips were made using the Intel 16nm CMOS process.

The class relies on open-source infrastructure:

- **Chisel** for parameterized RTL generation
  - Chisel adds hardware primitives to the Scala language
  - It has many libraries implementing common hardware components which can be reused and it compiles down to FIRRTL which emits synthesizable Verilog
- **Chipyard** frameowrk for SoC composition where CPU tiles, caches, accelerators, and peripherals are wired together using a diplomacy-based configuration approach
  - It has a collection of tools and generator libraries (such as cores)
  - Chip components are described modularly
  - Allows for simulation with commericial or open-source simulators
- **TileLink** as the on-chip interconnect protocol across caches, memory, accelerators, and MMIO devices.
- **Constellation** as a configurable NoC generator that can replace standard bus fabrics by routing TileLink traffic over a router-based network.
- **HAMMER** as a open-source flow tool that wraps physical design tools into standardized APIs
  - This helps abstract process specific concerns and have a programmable flow (Python) that can inject scripts/constraints into EDA tools and standardize signoff outputs

With this infrastructure, we are able to have a group of undergraduates learn the tools, design the chip, and have a tapeout all in the course of a semester. Students were divided into teams such as those working on accelerator RTL, peripherals, and Integration. I was part of the Integration group.

## Overview of Chips

### BearlyML'25

The first of the two digital chips made was BearlyML'25, a chip primarily focused on Machine Learning applications. The chip has two Saturn Tiles, each with a Saturn vector unit and an Outer Product Engine to compute matrix multiplications. In addition it also has a 2D Convolution accelerator to support convolution kernels. It also has a variety of communication protocols (I2C, SPI, UART, JTAG, etc) available.

Compared to the previous year, we opted to use 2 big cores (versus 4 smaller ones), and optimized them for vector operations and switched the architecture of them (from Rocket to Shuttle). Each core also has a TCM or tighly coupled memory bank which helps reduce reliance on L2. We compressed the number of accelerators and focused on optimizing both the Outer Product Engine and 2D Convolutoin accelerator. The Outer Product Engine was a completely new piece of RTL introduced this year. Another key system change was to move from a simple ring NoC to a SplitNoC approach where network traffic is split across two different topologies depending on the communication type, and further having a custom topology designed to improve memory bandwidth to the L2 banks. 

### DSP'25

The second of the two digital chips made was DSP'25, a chip primarily focused on audio and signal processing applications. The chip has two Saturn Tiles, each with a Saturn vector unit and 3 accelerators: a 1D Convolution accelerator, Wavelet Transform Engine, and a DMA (Direct Memory Access) Engine. The Wavelet Transform Engine was a completely new piece of RTL introduced this year.It has a variety of communication protocols (I2C, SPI, UART, JTAG, etc) available.

Compared to the previous, the DSP chip also opted for 2 cores (instead of 4), switch architectures, and optimzied pipelines in the Shuttle cores. The accelerators were improved upon and we also moved from the simple ring NoC to a SplitNoC approach. A custom topology was not used here, but rather we chose a bidirectional-torus topology.

## Integration Efforts

I was on the integration team, and helped coordinate the system-level integration, run benchmarks, go through the tapeout flow from RTL to GDSII. This included stems of synthesis, place and route, drc, lvs, and signoff. Both the chips met timing closure targets of 500MHz. 

We optimized area by experimenting with different floorplans, making sure to reduce congestion due to routing, place any neccesary obstructions or blockages, and inject any neccesary TCL scripts to be DRC and LVS clean.

## NoC Exploration

In previous years, the tapeout chips used an unidirectional ring NoC. However there was a lot of potential for performance to be gained here and I looked into extending the Constellation framework to design a custom topology that is synthesizable and fits cleanly into our chip's physical design.

By looking at a unidirectional ring topology and using benchmarks such as MNIST, I noticed that a huge bottleneck was the L2 cache. With it being on the opposite side of the chip as the cores, this resulted in the routers at the L2 banks being several hops away from either of the cores. A key motivation was trying to create a bypass or a shortcut between the cores and the L2 banks so that it is only a one-hop latency.

1) CustomTopology: Supporting Arbitrary Graph Topologies

Many standard topologies like mesh or ring are simple and efficient but can be suboptimal especially because they may introduce many many-hop delays due to the placement of specific nodes to one another. If we want maximize throughput between certain nodes (e.g. Core <-> L2, DMA <-> L2, DMA <-> PBUS, etc), it would be helpful to be able to travel between them in one hop.

The node layout in standard topologies is often very influenced by physical locality which may make certain connections many hops. I intrdouced CustomTopology in Constellation which in theory allows us to create any arbitrary graph (* though it still needs to pass diplomacy so there are likely some base requriements such as every node on the topology being able to be reached at least). CustoTopology simly helps us specify the number of nodes and exact edge connections for a defined topology.

By utilizing CustomTopology effectively, we can reduce latency of certain paths by drawing custom bypass or shortcut edges between nodes. In the chips we taped out we found it most useful to add or remove some edges from the unidirectional ring to minimize area cost while increasing performance.

2) Routing: Maintaining Deadlock Free Routing on Arbitrary Graphs

The creation of arbitrary graphs likely leads to increasing the number of cycles present, which leads to many more chances to deadlock. To make sure we are deadlock free, I introduced a few algorithms which can work for CustomTopology.

   * ``ShortestPathRouting``
     * This routing relation performs basic shortest-path routing between nodes. All flow paths are precomputed using BFS and stored as a next-hop map. During packet traversal, the router enforces transitions that match the expected next hop.
     * This strategy does not enforce deadlock freedom but can be used as a default routing relation and having a different deadlock-free algorithm as a fallback, when more virtual channels are available.
   * ``ShortestPathGeneralizedDatelineRouting``
     * This is a dateline-based, deadlock-free routing strategy for arbitrary graphs.
     * Deadlock is avoided by analyzing all shortest paths to:
       * 1) Build a channel dependency graph (CDG)
       * 2) Detect cycles
       * 3) Choose a set of cycle breaking edges and categorize them as "datelines"
       * 4) Every flow path is annotated with its VC transitions across datelines, and routers enforce those exact transitions
     * There are also some customizations available to adjust the VC injection scheme such as using a static VC, a hash algorithm, or leaving it up to the allocators.
     *  The number of VCs required is equal to the max number of dateline crossings + 1
   * ``CustomLayeredRouting``
     * This is a generalized deadlock-free routing strategy that assigns each flow to a unique layer (VC).
     * Algorithm Overview
       * 1) Compute all-pairs shortest paths
       * 2) Deduplicate identical edge paths
       * 3) Prune subpaths of longer routes
       * 4) Pack remaining flow paths into DAG layers (VCs)
       * 5) Each layer is enforced as a separate VC, which ensures each layer is acylic
     * The number of VCs required is equal to the number of packet layers and is determined automatically at compile time
     * This method can sometimes lead to more VCs than the dateline method for graphs but could be better for load balancing

These efforts were also pushed upstream to the Constellation repo: [https://github.com/ucb-bar/constellation](https://github.com/ucb-bar/constellation).

## Results and Impact

In the tapeout we also moved to a SplitNoC design where TileLink traffic is partitioned across two physically separate networks based on traffic type. In practice, this means we no longer force fundamentally different communication patterns to contend for the same links, routers, and buffering resources.

We have an ACD network which carries the performance critical traffic and a BE network that carries bulk/peripheral transfers. We optimized the ACD network using the CustomTopology and had the BE network be a standard undirectional ring. 

We benchmarked the NoC across various topologies and parameters (for number of VCs, buffer depth, flitSize, pipeline architecture) in terms of performance (on MNIST and other C tests) and area after going through the tapeout flow.

The following semester I participated in the the EE 194 bringup class as well: [Bringup of 16nm CMOS Chip](/project/bringup-16nm-cmos-chip).