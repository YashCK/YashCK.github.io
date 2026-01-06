


This wiki page focuses on introducing different NoC Configs and how to make use of the `CustomTopology` class and Routing Algorithms in Constellation to increase performance. 

For a more specific intro into Constellation you can check here [Constellation Primer](https://bwrcrepo.eecs.berkeley.edu/ee290c_ee194_intech22/sp25-chips/-/wikis/digital/integration/A-Quick-Primer-on-Constellation)

## Why Custom Topologies?

Many standard topologies like mesh or ring are simple and efficient but can be suboptimal especially because they may introduce many many-hop delays due to the placement of specific nodes to one another. If we want maximize throughput between certain nodes (e.g. Core <-> L2, DMA <-> L2, DMA <-> PBUS, etc), it would be helpful to be able to travel between them in one hop.

* The node layout in standard topologies is often very influenced by physical locality which may make certain connections many hops. 

By utilizing `CustomTopology` effectively, we can reduce latency of certain paths by drawing custom bypass or shortcut edges between nodes. In theory we could create any arbitrary graph but we found it most useful to most edges from a base unidirectional ring and add/remove certain edges to decrease latency of critical paths.

## Example

````scala
val baseRing = (0 until 8).map(i => TopologyEdge(i, (i + 1) % 8))
val bypasses = Seq((0, 4), (4, 0), (1, 5), (5, 1)).map { case (a, b) => TopologyEdge(a, b) }
val allEdges = baseRing ++ bypasses

val topology = CustomTopology(8, allEdges)
````

This is an example where we add two bidirectional edges, one between 0 and 4, and another between 1 and 5. Assume 0, 1 correspond to the Satrun Cores while 4, 5 correspond to the L2 caches. This would mean that we have two bidirectional edges between each Core and an L2 bank. 

## Importance of Routing Relations

The creation of arbitrary graphs likely leads to an increased number of cycles present in the underlying topology, and many more chances to deadlock. To make sure we are deadlock free, there are a few algorithms which can help.

#### ``ShortestPathRouting``

This routing relation performs basic shortest-path routing between nodes. All flow paths are precomputed using BFS and stored as a next-hop map. During packet traversal, the router enforces transitions that match the expected next hop.

This strategy does not enforce deadlock freedom. However in many simple test graphs it still seems to pass the deadlock tests. This needs to be investigated in more detail. Since it takes the shortest possible path without any other considerations, it is often the most efficient.

This Routing Relation can also be tested with different VC Allocators which may improve the load balancing across VCs, which in turns improves performance.

If using the PrioritizingVCAllocator, it is best to specify the `maxVCs` present in the instantiation: `ShortestPathRouting(maxVCs = 3)`. This helps assist the allocator in load balancing. 
* No strict VC count is required
* At least 1 VC per Virtual Subnetwork

This is best used in conjunction with `EscapeChannelRouting`. 

#### ``ShortestPathGeneralizedDatelineRouting``

This is a dateline-based, deadlock-free routing strategy for arbitrary graphs.

Deadlock is avoided by analyzing all shortest paths to:
1. Build a channel dependency graph (CDG)
2. Detect cycles
2. Choose a set of cycle breaking edges and categorize them as "datelines"
   - These act as VC transitions to break those cycles when routing
4. Every flow path is annotated with its VC transitions across datelines, and routers enforce those exact transitions
   - This means every hop must use the VC assigned to that edge

There is some capability to adjust the VC Injection scheme, this is which VC the router will decide to inject flows into at the start of the path. 
- `ShortestPathGeneralizedDatelineRouting(useStaticVC0 = false)`

If `useStaticVC0 = true`: all packets inject on VC 0 and increment VC on dateline edges
Else: hash-based injection from (a × src + b × dst) % numVCs

Note: The hashing scheme is purely to help us load balance the distribution of injection VCs randomly.

It is much more efficient to have useStaticVC0 = false (the default). However this can be useful to turn off to help debug if looking into how certain flows may be handled. 

This is a deadlock free routing relation. The number of VCs required are automatically computed.
* Equal to the max number of dateline crossings + 1 (e.g., usually 2–5 VCs)

**NOTE:** If the use of this algorithm does not allow the NoC config to elaborate, it is likely that there were not enough VCs provided. The required number of VCs will be present in the chisel.log file. 

#### ``CustomLayeredRouting``

This is a generalized deadlock-free routing strategy that assigns each flow to a unique layer (VC).

Algorithm Overview:
1. Compute all-pairs shortest paths
2. Deduplicate identical edge paths
3. Prune subpaths of longer routes
4. Pack remaining flow paths into DAG layers (VCs)
4. Each layer is enforced as a separate VC, which ensures each layer is acylic

VC Requirements: Equal to number of packed layers
* This is determined automatically at compile time.  
* However this method often leads to a few more VCs than dateline methods for graphs. There are still cases where this performs better in terms of having fewer VCs required though.

**NOTE: ** If the use of this algorithm does not allow the NoC config to elaborate, it is likely that there were not enough VCs provided. The required number of VCs will be present in the chisel.log file. 

## EscapeChannelRouting with Custom Topology

When using a non-deadlock-free algorithm like `ShortestPathRouting`, we can wrap it with an escape route:

````scala
EscapeChannelRouting(
  escapeRouter    = ShortestPathGeneralizedDatelineRouting(),
  normalRouter    = ShortestPathRouting(maxVCs = 2),
  nEscapeChannels = 3
)
````

This allocates a fixed number of VCs for the guaranteed-safe path (via dateline routing), allowing the remaining VCs to the faster routing approach. However depending on the VCs required for being deadlock safe, it may not be feasible enough to have other VCs for a non-deadlock free algorithm. 

## Walkthrough Examples

### Bearly25 Config

The current BearlyConfig uses a split topology to support high-priority traffic (ACD) on a customized ring and low-priority traffic (BE) on a separate torus.

````scala
new constellation.soc.WithSbusNoC(constellation.protocol.SplitACDxBETLNoCParams(
  constellation.protocol.DiplomaticNetworkNodeMapping(
    inNodeMapping = ListMap(
      "Core 0" -> 1,
      "Core 1" -> 2,
      "serial_tl" -> 9,
      "bearly-near-mem-conv-read[0]" -> 5,
      "bearly-near-mem-conv-write[0]" -> 5
    ),
    outNodeMapping = ListMap(
      "serdesser[2]" -> 8,
      "serdesser[1]" -> 7,
      "serdesser[0]" -> 6,
      "Core 0" -> 0,
      "Core 1" -> 3,
      "ram[0]" -> 4,
      "pbus" -> 9
    )
  ),
  acdNoCParams = {
    val nNodes = 10
    val originalEdges = Seq((0,1), (1,2), (2,3), (3,4), (4,5), (5,6), (7,8), (8,9), (9,0)).map { case (a,b) => TopologyEdge(a,b) }
    val bypassEdges = Seq((1,7), (7,1), (6,2), (2,6), (2,1)).map { case (a,b) => TopologyEdge(a,b) }
    val allEdges = originalEdges ++ bypassEdges
    NoCParams(
      topology = CustomTopology(nNodes, allEdges),
      channelParamGen = (a, b) => UserChannelParams(Seq.fill(9) { UserVirtualChannelParams(4) }),
      routingRelation = NonblockingVirtualSubnetworksRouting(ShortestPathGeneralizedDatelineRouting(), 3, 3),
      routerParams = _ => UserRouterParams(
        combineRCVA = false,
        combineSAST = false,
        coupleSAVA  = false
      )
    )
  },
  beNoCParams = NoCParams(
    topology = UnidirectionalTorus1D(10),
    channelParamGen = (a, b) => UserChannelParams(Seq.fill(4) { UserVirtualChannelParams(2) }),
    routingRelation = NonblockingVirtualSubnetworksRouting(UnidirectionalTorus1DDatelineRouting(), 2, 2)
  )
))++
````

### Understanding SplitACDxBETLNoC

This SplitNoC model splits the ABCDE TileLink channel messages into cache-coherent and non-coherent traffic and allows us to optimize each portion independently. This means we can have different topologies, routing relations, virtual channels, buffer depths, and more for each one. 
- ACD traffic is typically more latency-sensitive and performance-critical. It gets prioritized by allocating more VCs, better routing (like dateline or layered), and customized topologies.
- BE traffic is routed on a separate, lower-bandwidth torus to avoid impacting performance-critical cores. It typically has a lot less area as well.

The SplitNoC helps us save on area as well as timing. There are 3 virtual subnetworks for the 3 channels ACD and 2 virtual subnetworks for 2 channels BE. 

The routing pipeling flags are not enabled on this config. 

![Screenshot_2025-05-13_at_9.07.54_PM](uploads/80eac978ab835d75259ba6a602163ff1/Screenshot_2025-05-13_at_9.07.54_PM.png)

The bypasses included are bidirectional edges between each core and L2, as well as between cores. This allows the TCMs to talk to one another. We also remove the edge between the L2 banks in the ACD topology for better timing and area closure.

## Pipelining

Constellation’s router generators have a few different tuning options available. 

There are typically 4 stages.
1. RC stage: The head flit of the packet queries the router’s RouteComputer to determine the next set candidate virtual channels it may allocate.
2. VA stage: The head flit of the packet queries the router’s VirtualChannelAllocator to allocate a virtual channel from the candidate set of next virtual channels.
3. SA stage: Flits ask the SwitchAllocator for access to the crossbar switch in the router. This stage also checks that the next virtual channel has an empty buffer slot to accomodate this flit.
4. ST stage: A flit traverses the crossbar switch. 

![router.svg](uploads/1cc3791e61e3ca6c14e6581e988cc9cd/router.svg)

The standard pipelineis a 4-hop router, with these 4 as  separate stages. Two flags exist in the base router generator to reduce the hop count.
* `combineRCVA` performs route-compute and virtual-channel-allocation in the same cycle
   * This is best on simpler routing policies.
* `combineSAST` performs switch allocation and switch traversal in the same cycle.
   * This is best for low-radix routers

![pipeline_base.svg](uploads/7d83982af5c01a5fb2c62518d66815d6/pipeline_base.svg)

In the base architecture, stalls can occur due to a delay in reallocating an output virtual channel to a new packet. 
* Enabling `coupleSAVA` allows the freed virtual channel to be immediately made available on the same cycle. However coupleSAVA can introduce long combinational paths on high-radix routers.

![credit_stall.svg](uploads/f0bf4c8042fd0cae59e1818e099f2c3f/credit_stall.svg)


## Some Notes on Performance

The rule of thumb is that shorter paths are faster.

VC depth: Insufficient VCs lead to stalls and head-of-line blocking.

Congestion: Central links or shared routers slow down under load.

Routing adaptivity: Static paths may under-utilize network resources.

Strategies for Performance Gains
1. Add bypass links in critical traffic patterns.
2. Use EscapeChannelRouting to support adaptivity without compromising safety.
3. Tune combineRCVA / combineSAST to collapse pipeline stages (1–2 cycles saved per flow).
3. Make sure to have benchmarks and applications to help evaluate and compare configA vs. configB.

When benchmarking it would be useful to compare cycle counts after sweeping different buffer size, virutal channels, routing algorithms, and topologies.

## Physical Design Implications

While the CustomTopology is great for improving performance in simulation, it needs to be physically realizable as well. Adding shortcuts between Cores and L2 is very doable, but if the topology has an extreme number of edges it can very difficult for tools to optimize and use the floorplan effectively. 

In conjunction with the CustomTopology, it would be useful to reorder nodes as necessary to help reduce delays and optimize the Config. 

For example if we PAR a config which represents a unidirectional ring + crossbar between cores and L2, most of the center of the chip would be designated just for the NoC. The inclusion of accelerators and other components are important to take into consideration. 

In addition, if turning on the flags to pipeline the routing stages, it can lead to long combinational paths which would influence the critical path delay.

VC buffer width or more VCs ==> more queues and wires ==> more area and congestion

In general for timing, it is difficult to narrow down what is causing the tool to create long critical paths all the time. Many factors such as just adjusting the buffer width from 4 to 2 can have a drastic effect on timing, even if the area saved is really small.