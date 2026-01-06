

## Intro and NoC Configuration

Constellation is a NoC (Network-on-Chip) framework that helps enable interconnect design in Chisel-based SoCs.

This is a condensed version/notes but the official documentation can be found here: [Documentation](https://constellation.readthedocs.io/en/latest/index.html)

There are 3 main components
1. Physical Specification – defines topology and router/channel microarchitecture
2. Flow Specification – defines what communication patterns occur
3. Routing Specification – defines how flows are mapped onto the physical topology

Example:
````scala
nocParams = NoCParams(
    topology = BidirectionalLine(5),
    channelParamGen = (a, b) => UserChannelParams(Seq.fill(5) { UserVirtualChannelParams(4) }),
    routingRelation = NonblockingVirtualSubnetworksRouting(BidirectionalLineRouting(), 5, 1)
)

````

## Physical Specification

This is the portion which defines the shape and behavior of the network. 

|Field              |Purpose                                                                     |
|-------------------|----------------------------------------------------------------------------|
|topology           |Defines the graph of routers and their connectivity (via PhysicalTopology)  |
|channelParamGen    |A generator for channel-level microarchitecture (e.g., flit width, buffers) |
|ingresses, egresses|Points where traffic enters and exits the NoC                               |
|routerParams       |Specifies parameters for each router node (e.g., number of virtual channels)|

<details>
<summary>More Details on Topologies</summary>

### Topologies

These define how routers are connected. A topology should extend the `PhysicalTopology` trait. 

````scala
trait PhysicalTopology {
  val nNodes: Int
  def topo(src: Int, dst: Int): Boolean
  val plotter: PhysicalTopologyPlotter
}
````

One specific example is the UnidirectionalTorus1D (one way ring):
````scala
case class UnidirectionalTorus1D(n: Int) extends Torus1DLikeTopology {
  val nNodes = n
  def topo(src: Int, dst: Int): Boolean = (dst - src + nNodes) % nNodes == 1
}
````

|Name               |Description                                                                 |Parameters  |
|-------------------|----------------------------------------------------------------------------|------------|
|UnidirectionalLine(n)|Linear chain of n nodes with one-way links                                  |nNodes      |
|BidirectionalLine(n)|Same as above, but links in both directions                                 |nNodes      |
|UnidirectionalTorus1D(n)|Ring network                                                                |nNodes      |
|BidirectionalTorus1D(n)|Ring with links in both directions                                          |nNodes      |
|Butterfly(k, n)    |Multistage k-ary n-fly topology                                             |kAry, nFly  |
|BidirectionalTree(h, d)|Tree with branching                                                         |height, dAry|
|Mesh2D(nX, nY)     |2D mesh grid                                                                |nX, nY      |
|UnidirectionalTorus2D(nX, nY)|2D torus                                                                    |nX, nY      |
|BidirectionalTorus2D(nX, nY)|2D torus with bidirectional links                                           |nX, nY      |

We can also wrap a base topology with a TerminalRouter. This can help lower the number of input/output ports present on a router. 

Another strategy that can be done is creating a `HierarchicalTopology`. This composes a large network out of smaller subnetworks.

````scala
HierarchicalTopology(
  base = UnidirectionalTorus1D(4),
  children = Seq(
    HierarchicalSubTopology(0, 2, BidirectionalLine(5)),
    HierarchicalSubTopology(2, 1, BidirectionalLine(3))
  )
)
````
* This connects child subgraphs to parent routers
* A HierarchicalRouting relation must be used
* Can be combined with TerminalRouter

In order to write your own Topology:
1. Create a new case class that extends PhysicalTopology
2. Implement nNodes, topo(src, dst) [and plotter (optional)]
3. Plug into NoCParams(topology = YourNewTopology(n))

Example:

````scala
case class StarTopology(n: Int) extends PhysicalTopology {
  val nNodes = n
  def topo(src: Int, dst: Int): Boolean = {
    (src == 0 && dst != 0) || (dst == 0 && src != 0)
  }
  val plotter = new BasicTopologyPlotter(n) // optional visualization
}
````
* This should link from one center node to all nodes
* It reverses links back to the center
* There are no links between non-central nodes

This could be used in the following way:

````scala
val starParams = NoCParams(
  topology = StarTopology(5),
  ...
)
````

![Screenshot_2025-04-18_at_8.06.15_PM](uploads/b50e37df774c75d2ebadd257b449cca9/Screenshot_2025-04-18_at_8.06.15_PM.png)

When creating or working with a topology it is best to consider some of the following:
- What is the communication pattern?
  - Hierarchical vs Flat connectivity
  - If certain sections are heavier or require bidirectional connection, then that should be taken into account
- Is bandwidth uniform across nodes?
- How will blocks connect to the NoC?
  - Do they connect directly or via TerminalRouters, etc

</details>

<details><summary>More Details on Channels</summary>

Channels are the unidirectional links which connect routers in your physical topology.
This behavior is controlled in the following line
```channelParamGen: (src: Int, dst: Int) => UserChannelParams```

This function is called once for every directed edge in the topology.

The `UserChannelParams` class:

case class UserChannelParams(
  virtualChannelParams: Seq[UserVirtualChannelParams],
  channelGen: Parameters => ChannelOutwardNode => ChannelOutwardNode,
  crossingType: ClockCrossingType,
  useOutputQueues: Boolean,
  unifiedBuffer: Boolean,
  srcSpeedup: Int,
  destSpeedup: Int
)

|Field              |Use                                                                         |Possible Use|
|-------------------|----------------------------------------------------------------------------|------------|
|virtualChannelParams|List of VCs and their buffer depths                                         |More VCs could reduce head-of-line blocking, but would increase area|
|channelGen         |Optional logic to add pipeline stages (e.g., delay buffers)                 |1-2 FIFOs stages could be added to break long combinational paths|
|crossingType       |Clock domain crossing                                                       |don't use   |
|srcSpeedup / destSpeedup|Controls how many flits enter/exit per cycle                                |If your source/destination can issue faster then can make it >1|
|useOutputQueues    |Whether to include output buffering                                         |if very area constrained that could make it False|
|unifiedBuffer      |Whether input/output use same storage                                       |Unified buffers can save area, but maybe be less flexible|

#### Virtual Channels

These can help avoid head-of-line blocking by allowing multiple logical flows to share a physical link.

Head-of-line blocking is a possible bottleneck where one flow of packets blocks other behind it, even though they could have gone forward.
* For example if you have a router input port with only one queue and no VCs
   * Input Queue:
[ Packet A (dest 5) ]
[ Packet B (dest 3) ]
* If Packet A can’t move forward (maybe its route is congested), then Packet B is also stuck — even though its route could be free
* Virtual Channels would allow you to split a physical link into multiple logical queues:
   * VC 0: [ Packet A (dest 5) ]
VC 1: [ Packet B (dest 3) ]
* 

A flit is a flow control digit. It is the smallest unit of data that can be routed through the NoC at one time. If the NoC is received low-flit packets then 1VC should be fine. However if that is not the case, then you may want to have 2+ VCs to avoid stalls. Each VC does add more area.

Optional Speedup Fields: 
* `srcSpeedup`: How many flits enter per cycle
* `destSpeedup`: How many flits exit per cycle
These are only really useful if the src/dest modules accept flits faster than the default NoC bandwidth
- If you set srcSpeedup > destSpeedup it will probably lead to backpressure and starvation which is bad

</details>

<details><summary>More Details on Terminals</summary>

Terminals are how you attach modules like RocketTiles, DMAs, etc. to the NoC.

These can be defined via:
````scala
ingresses = Seq(UserIngressParams(nodeId = 0, payloadBits = 128)),
egresses  = Seq(UserEgressParams(nodeId = 5, payloadBits = 128))
````

|Field              |Use                                                                         |
|-------------------|----------------------------------------------------------------------------|
|nodeId             |Which router this terminal connects to                                      |
|payloadBits        |Width of data payload (flit size)                                           |
|egressId           |Optional ID if multiple egresses at same node                               |

It's best to terminal and router payloadBits aligned. If you don't Constellation may use some width converters which could cause problems.

</details>

<details><summary>More Details on Routers</summary>

A router is a hardware module that receives flits (flow control digits) and determines:
* Where to send them next (routing)
* When they’re allowed to go (scheduling and flow control)
* How to avoid blocking/flit loss (buffering, virtual channels)

This is an important part of packet forwarding and for managing traffic
* It can be thought of as a traffic intersection with multiple input lanes (channels), where each lane can have virtual sub-lanes (VCs). At every clock cycle, it decides which flit goes through what direction so that we can avoid deadlock/congestion.


Routers have 4 pipeline stages. When a packet (group of flits) reaches a router, the head flit is decoded and the router performs a pipeline of decisions:

|Stage              |Name                                                                        |What It Does                                                         |
|-------------------|----------------------------------------------------------------------------|---------------------------------------------------------------------|
|RC                 |Route Compute                                                               |Decides the possible next hops (based on topology & routing relation)|
|VA                 |Virtual Channel Allocation                                                  |Picks an available virtual channel on the outgoing link              |
|SA                 |Switch Allocation                                                           |Picks who gets to use the crossbar this cycle                        |
|ST                 |Switch Traversal                                                            |Flit travels across the crossbar to output port                      |

Once the flit traverses (ST), it goes to the next router or terminal.

This part is important because it can affect the congestion/traffic flow. Somethings for consideration are:
1. If there are many timing violations or long paths, then can potentially use slower routers (more stages), or a lot more buffers
2. If there are too few virtual channels -> we get head-of-line (HOL) blocking
3. If we have a bad allocator -> we get starvation (especially on asymmetric traffic)
4. Larger buffer sizes = more SRAM
5. More VCs = wider muxes, more registers
6. High-speed allocators = larger combinational logic

All flow control is credit based. Here is how it can be customized: 
````scala
UserRouterParams(
  nVirtualChannels = 2,
  payloadBits = 64,
  bufferSize = 4,
  combineRCVA = true,
  combineSAST = false,
  coupleSAVA = true,
  vcAllocator = new ISLIPMultiVCAllocator()
)
````

|Param              |Meaning                                                                     |Tradeoff                                                             |
|-------------------|----------------------------------------------------------------------------|---------------------------------------------------------------------|
|nVirtualChannels   |Number of VCs per input port                                                |More = better utilization, less blocking. But bigger area.           |
|payloadBits        |Size of each flit in bits                                                   |Match with terminal/channel to avoid width converters                |
|bufferSize         |Entries per VC buffer                                                       |Longer = fewer stalls but more SRAM                                  |
|combineRCVA        |Combine RC and VA                                                           |Lowers latency, but longer timing path                               |
|combineSAST        |Combine SA and ST                                                           |Fewer stages; good for small routers                                 |
|coupleSAVA         |Free VC reused instantly after ST                                           |Faster VC turnaround but longer critical path                        |
|vcAllocator        |VC allocation policy                                                        |Affects fairness vs throughput                                       |

We can also control how VCs show be granted especially under contention:

|Allocator          |Behavior                                                                    |When to Use                                                          |
|-------------------|----------------------------------------------------------------------------|---------------------------------------------------------------------|
|PIMMultiVCAllocator|Parallel iterative matching (good throughput)                               |General high-load NoCs                                               |
|ISLIPMultiVCAllocator|iSLIP matching (fair, round-robin-like)                                     |Multi-core fairness                                                  |
|RotatingSingleVCAllocator|One VC per cycle, rotates                                                   |Low-resource, low-contention                                         |
|PrioritizingSingleVCAllocator|Single VC, priority-based                                                   |Specialized flows (e.g., control path > DMA)                         |

![Screenshot_2025-04-18_at_8.50.03_PM](uploads/6c6b42a9b4ffc174953469ca727fa8f2/Screenshot_2025-04-18_at_8.50.03_PM.png)

</details>

## Flow Specification

A flow in Constellation defines an allowed expected source → destination path in the NoC. It connects:
- An ingress terminal (the source IP block)
- An egress terminal (the destination IP block)
- A virtual subnetwork (vNet) used to carry that flow

````scala
case class FlowParams(
  ingressId: Int,    // index in ingresses
  egressId: Int,     // index in egresses
  vNetId: Int,       // which virtual subnetwork
  fifo: Boolean = false
)
````

Technically a NoC can allow all-to-all communication but that would be overkill a ton of area, so we can generate more optimized hardware so that we only focus on the flows we know the system will use.

* Fewer declared flows → smaller crossbars, buffers, and control logic.

<details><summary>Details on Virtual Subnetworks (vNets)</summary>

A vNet is a separate lane in the NoC — flows on different vNets don’t interfere with each other. 
We can assign flows to vNets to separate traffic classes, prevent deadlock, and improve fairness or priority.

Example:
````scala
FlowParams(ingressId = 0, egressId = 3, vNetId = 0)  // e.g., DMA
FlowParams(ingressId = 1, egressId = 3, vNetId = 1)  // e.g., control
````

There is a function which defines blocking between vNets:

```vNetBlocking: (blocker: Int, blockee: Int) => Boolean```

If `vNetBlocking(x, y) == true`, then vNet x must be able to make forward progress even when vNet y is stalled.

</details>

## Routing Specification

Constellation lets you define a RoutingRelation that governs:
1. Which links/VCs a packet is allowed to traverse
2. Which paths are legal for which flows
3. What priorities to assign during allocation

In order to make these decisions the router knows `ChannelRoutingInfo` which is the metadata for channels.

````scala
ChannelRoutingInfo(src = 1, dst = 2, vc = 0, n_vc = 2)
````

|Field              |Meaning                                                                     |
|-------------------|----------------------------------------------------------------------------|
|src, dst           |Physical node IDs                                                           |
|vc                 |Virtual channel index                                                       |
|n_vc               |Total number of VCs                                                         |

Constellation will match the following against routing rules to determine which paths are valid.
````scala
FlowRoutingInfo(
  ingressId = 0,
  egressId = 2,
  vNetId = 1,
  ingressNode = 0,
  egressNode = 4,
  ...
)
````

<details><summary>More Details on Routing Relation</summary>

Routing Relation is an abstract class which we can implement. One key function to take note of is:
- ```def rel(src: ChannelRoutingInfo, nxt: ChannelRoutingInfo, flow: FlowRoutingInfo): Boolean```

  - Returns true if it's legal for flow to move from src to nxt channel
  - Other functions of Routing Relation
    - `isEscape()` → for escape channel routing 
    - `getPrio()` → sets allocation priority

</details>

<details><summary>More Details on Prebuilt Routing Policies</summary>

Constellation gives you prebuilt routing policies, depending on your topology. 

|RoutingRelation    |Topology Target                                                             |Routing Behavior                              |When to Use                                                                                                                 |
|-------------------|----------------------------------------------------------------------------|----------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
|AllLegalRouting()  |Any (toy or test nets)                                                      |Every channel/VC is legal                     |No deadlock prevention, poor synthesis, seems quite bad                                                                     |
|UnidirectionalLineRouting()|UnidirectionalLine(n)                                                       |Fixed forward direction (node i → i+1)        |Simple and deadlock-free                                                                                                    |
|BidirectionalLineRouting()|BidirectionalLine(n)                                                        |Picks shortest path (either direction)        |Direction chosen dynamically, It May require VC separation to avoid deadlock loops                                          |
|UnidirectionalTorus1DDatelineRouting()|UnidirectionalTorus1D(n)                                                    |Minimal routing around ring; avoids dateline  |This requires ≥2 VCs. Dateline concept breaks cycles → avoids deadlock.                                                     |
|BidirectionalTorus1DRandomRouting()|BidirectionalTorus1D(n)                                                     |Randomly chooses direction (cw/ccw) at ingress|Randomized for load balancing. It requires ≥2 VCs. It avoids persistent congestion in uniform loads.                        |
|BidirectionalTorus1DShortestRouting()|BidirectionalTorus1D(n)                                                     |Always picks direction with fewer hops        |This is smarter than random but still needs dateline logic and ≥2 VCs                                                       |
|ButterflyRouting() |Butterfly(k, n)                                                             |Static dimension-ordered across stages        |Butterfly structured topology used in parallel FFTs, radix-n networks                                                       |
|BidirectionalTreeRouting()|BidirectionalTree(h, d)                                                     |Routes up to common ancestor, then down       |Good for multicast/aggregation trees (e.g., mem hierarchy). It is always deadlock-free.                                     |
|Mesh2DDimensionOrderedRouting()|Mesh2D(nx, ny)                                                              |First X → then Y (XY routing)                 |This is deterministic routing. No deadlock if used with 1 VC per dim or VC separation.                                      |
|Mesh2DWestFirstRouting()|Mesh2D(nx, ny)                                                              |Route westward first (then any other dir)     |Restricts early direction → breaks cyclic dependences. Best for west-edge-heavy traffic.                                    |
|Mesh2DNorthLastRouting()|Mesh2D(nx, ny)                                                              |Route any dir except north first              |Helps avoid deadlock when northward traffic can wait                                                                        |
|Mesh2DEscapeRouting()|Mesh2D(nx, ny)                                                              |Adaptive routing w/ fallback escape VC        |Uses deterministic XY for escape; others route freely. Needs ≥2 VCs. The main tradeoff is between adaptivity and complexity.|
|DimensionOrderedUnidirectionalTorus2DRouting()|UnidirectionalTorus2D(nx, ny)                                               |X-first routing around torus                  |This assumes ring-wrap in each dimension. Needs dateline or careful VC planning.                                            |
|DimensionOrderedBidirectionalTorus2DRouting()|BidirectionalTorus2D(nx, ny)                                                |X-first bidirectional routing                 |Picks minimal direction in X, then in Y. Must use ≥2 VCs with dateline logic. Best when wrapping is needed.                 |

</details>

Some other notes which are bit complicated are Escape Channel Routing:
* this uses adaptive routing but still tries to be deadlock-free
* Packets try adaptive routes (VCs 1..N) but can fallback to escape route (VC 0) if needed
* Escape channels should not depend on normal channels, otherwise there can be a circular wait

There are also Virtual Subnetwork Routing Policies to control inter-vNet behavior.

## Protocol Notes

The NoC we define in Constellation is just a generic message-passing interconnect. But Chipyard modules (RocketTile, DMA, Memory) speak AXI4 or TileLink which are not generic flits.

That means we need a layer to:
* Wrap TL/AXI transactions into NoC packets
* Map protocol masters/slaves to NoC ingress/egress terminals
* Guarantee that protocol semantics (like atomicity or ordering) are preserved

This is handled by a ProtocolParams wrapper, which:
* Defines the payload size, vNets, and flows
* Connects protocol edges to NoC terminals
* Provides Chisel IO bundles for SoC elaboration

````scala
trait ProtocolParams {
  val minPayloadWidth: Int
  val ingressNodes: Seq[Int]
  val egressNodes: Seq[Int]
  val nVirtualNetworks: Int
  val vNetBlocking: (Int, Int) => Boolean
  val flows: Seq[FlowParams]
  def genIO(): Data
  def interface(terminals: NoCTerminalIO, ..., protocol: Data): Unit
}
````

|Field              |Purpose                                                                     |TipNotes                                      |
|-------------------|----------------------------------------------------------------------------|----------------------------------------------|
|minPayloadWidth    |Minimum flit width needed                                                   |Typically ≥ 64 bits for TL                    |
|ingressNodes       |Which NoC nodes host masters                                                |Each entry = index in your topology           |
|egressNodes        |Which NoC nodes host slaves                                                 |Same as above                                 |
|nVirtualNetworks   |One per TL channel (A,B,C,D,E)                                              |TL-C = 5, TL-UL = 2                           |
|vNetBlocking       |Blocking relation among TL channels                                         |Used to prioritize certain channel progress   |
|flows              |List of expected master→slave flows                                         |Enables RTL pruning                           |
|genIO()            |Returns IO bundle of the full protocol                                      |e.g., TLBundle or AXI4Bundle                  |
|interface()        |Hook for connecting TL ↔ NoC terminals                                      |You define this for custom adapters           |


Example:

````scala
TileLinkABCDEProtocolParams(
  edgesIn = Seq(master1Edge, master2Edge),
  edgesOut = Seq(slave1Edge, memEdge),
  edgeInNodes = Seq(0, 1),
  edgeOutNodes = Seq(2, 3)
)
````
* This create 2 ingresses at nodes 0 and 1
* creates 2 egresses at nodes 2 and 3
* Declare flows from 0->1 and 2->3
* Use 5 virtual networks for A/B/C/D/E channels

TileLink has multiple independent channels, each with different directionality and ordering rules:
* A (requests), D (responses)
* B/C/E (coherence)

Each vNet must avoid deadlock and maintain protocol semantics. 
For example:
- vNet 0 → channel A
- vNet 1 → channel D
- vNet 2 → channel B

## Tapeout Example

<details><summary>Mbus Instantiation</summary>

````scala
new constellation.soc.WithMbusNoC(
    tlnocParams = constellation.protocol.SimpleTLNoCParams(
      nodeMappings = constellation.protocol.DiplomaticNetworkNodeMapping(
        inNodeMapping = ListMap(
          "L2 InclusiveCache[0]" -> 0,
          "L2 InclusiveCache[1]" -> 1,
          "L2 InclusiveCache[2]" -> 2,
          "L2 InclusiveCache[3]" -> 3
        ),
        outNodeMapping = ListMap(
          "serdesser[0]" -> 4
          // "ram[0]" -> 5
        )
      ),
      nocParams = NoCParams(
        topology = BidirectionalLine(5),
        channelParamGen = (a, b) => UserChannelParams(Seq.fill(5) { UserVirtualChannelParams(4) }),
        routingRelation = NonblockingVirtualSubnetworksRouting(BidirectionalLineRouting(), 5, 1)
      )
    )
  ) ++
````

|Feature            |Value                                                                       |Explanation                                   |
|-------------------|----------------------------------------------------------------------------|----------------------------------------------|
|Topology           |BidirectionalLine(5)                                                        |A 5-node line. Simple and area-efficient.     |
|In Nodes           |L2 caches [0–3] → nodes 0–3                                                 |Each cache has its own ingress router.        |
|Out Node           |SerDes → node 4                                                             |Likely where data leaves chip or enters another domain.|
|VCs                |5 VCs per channel, 4 buffers each                                           |Plenty of buffering for congestion tolerance. |
|Routing            |NonblockingVirtualSubnetworksRouting(...)                                   |Ensures vNet isolation; 1 escape VC prevents deadlock.|

</details>

<details><summary>Sbus Instantiation</summary>

Sbus Instantiation: 
````scala
  new constellation.soc.WithSbusNoC(
    tlnocParams = constellation.protocol.SimpleTLNoCParams(
      nodeMappings = constellation.protocol.DiplomaticNetworkNodeMapping(
        // Core * corresponds to each RocketTile
        // serial_tl corresponds to the serial-tl port
        inNodeMapping = ListMap(
          "Core 0" -> 0,
          "Core 1" -> 1,
          "serial_tl" -> 3,
          "DMA-Tile[0]" -> 8
          // "" -> 8
        ),
        // serdesser corresponds to SBUS-facing L2 cache ports (TODO: verify)
        // pbus corresponds to the peripheral bus
        // TSI is on the pbus, so serial-tl and pbus should be on the same node
        outNodeMapping = ListMap(
          "serdesser[3]" -> 7,
          "serdesser[2]" -> 6,
          "serdesser[1]" -> 5,
          "serdesser[0]" -> 4,
          "ram[0]" -> 2,
          "pbus" -> 3
        )
      ),
      nocParams = NoCParams(
        topology = UnidirectionalTorus1D(9),
        // routerParams = (i) => UserRouterParams(payloadBits = 128),
        channelParamGen = (a, b) => UserChannelParams(Seq.fill(10) { UserVirtualChannelParams(4) }),
        // channelParamGen = (a, b) => UserChannelParams(Seq.fill(4) { UserVirtualChannelParams(4) }),
        routingRelation = NonblockingVirtualSubnetworksRouting(UnidirectionalTorus1DDatelineRouting(), 5, 2)
      )
    ),
    inlineNoC = false
  ) ++
````

|Feature            |Value                                                                       |Explanation                                   |
|-------------------|----------------------------------------------------------------------------|----------------------------------------------|
|Topology           |UnidirectionalTorus1D(9)                                                    |A ring with 9 nodes, wraparound enabled.      |
|In Nodes           |Cores, DMA, serial-tl                                                       |Each source (e.g., RocketTile or DMA) maps to a node.|
|Out Nodes          |L2 (SBUS side), MMIO, RAM, pbus                                             |These handle replies or memory/coherent transactions.|
|VCs                |10 VCs per channel, 4 buffers each                                          |Heavily buffered for bursty traffic.          |
|Routing            |NonblockingVirtualSubnetworksRouting(UnidirectionalTorus1DDatelineRouting(), 5, 2)|Adaptive, escape-channeled ring routing. Ensures full throughput and deadlock-free behavior.|
|inlineNoC          |FALSE                                                                       |Physically separates the NoC hierarchy from surrounding logic — important for floorplanning.|
</details>

## Other

You can run RTL simulations of the NoC Config using a test harness:
Visit this page for more info: [Running Tests](https://constellation.readthedocs.io/en/latest/Introduction/SimpleTests.html)