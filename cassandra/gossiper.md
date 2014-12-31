# Gossiper
The gossiper is responsible for making sure every node in the system eventually knows important information about every other node's state, including those that are unreachable or not yet in the cluster when any given state change occurs.

##API

Information to gossip is wrapped in an ApplicationState object, which is essentially a key/value pair. (See "Data Structures" below for more detail.) The gossiper propagates these to other nodes, where interested classes subscribe to changes via the `IEndPointStateChangeSubscriber` interface. This provides onRemove, onJoin, onAlive, and onDead methods indicating the obvious things, and onChange for ApplicationState changes. onChange is called once for each ApplicationState. There are two non-obvious properties to this:
1. If a node makes multiple changes to a given ApplicationState key, other nodes are guaranteed to see the most recent one but not intermediate ones
2. here is no provision for deleting an ApplicationState entirely

##Gossiper implementation
Gossip timer task runs every second. During each of these runs the node initiates gossip exchange according to following rules:
1. Gossip to random live endpoint (if any)
2. Gossip to random unreachable endpoint with certain probability depending on number of unreachable and live nodes
3. If the node gossiped to at (1) was not seed, or the number of live nodes is less than number of seeds, gossip to random seed with certain probability depending on number of unreachable, seed and live nodes.

These rules were developed to ensure that if the network is up, all nodes will eventually know about all other nodes. (Clearly, if each node only contacts one seed and then gossips only to random nodes it knows about, you can have partitions when there are multiple seeds -- each seed will only know about a subset of the nodes in the cluster. Step 3 avoids this and more subtle problems.)

This way a node initiates gossip exchange with one to three nodes every round (or zero if it is alone in the cluster)

##Data structures

HeartBeatState

Consists of generation and version number. Generation stays the same when server is running and grows every time the node is started. Used for distinguishing state information before and after a node restart. Version number is shared with application states and guarantees ordering. Each node has one HeartBeatState associated with it.

ApplicationState

Consists of state and version number and represents a state of single "component" or "element" within Cassandra. For instance application state for "load information" could be (5.2, 45), which means that node load is 5.2 at version 45.  Version number is shared by application states and HeartBeatState to guarantee ordering and can only grow.

EndPointState

Includes all ApplicationStates and HeartBeatState for certain endpoint (node). EndPointState can include only one of each type of ApplicationState, so if EndPointState already includes, say, load information, new load information will overwrite the old one. ApplicationState version number guarantees that old value will not overwrite new one.

endPointStateMap

`ConcurrentHashMap<InetAddress, EndpointState>()`

Internal structure in Gossiper that has EndPointState for all nodes (including itself) that it has heard about.

##Gossip Exchange
###GossipDigestSynMessage
Node starting gossip exchange sends GossipDigestSynMessage, which includes a list of gossip digests. A single gossip digest consists of endpoint address, generation number and maximum version that has been seen for the endpoint. In this context, maximum version number is the biggest version number in EndPointState for this endpoint. An example to illustrate this better:

Suppose that node 10.0.0.1 has following information in its endPointStateMap (remember that endPointStateMap includes also node itself):

```
EndPointState 10.0.0.1
  HeartBeatState: generation 2, version 541662
  ApplicationState "STATUS": NORMAL,-5125166994968203647,16
  ApplicationState: HOST_ID: 054040af-7998-4ae4-861b-28edefe7e64e,2
  ApplicationState: NET_VERSION: 9,1
  ApplicationState: RELEASE_VERSION: 3.0.0-SNAPSHOT,4
  ApplicationState: RPC_ADDRESS: 10.0.0.1,3
  ApplicationState: TOKENS: -5125166994968203647,-6296082587704969101,15
  ApplicationState: LOAD: 182792,45
  ApplicationState: DC: datacenter1,6
  ApplicationState: RACK: rack1,8
  ApplicationState: SEVERITY: 0.2557544708251953,541661
EndPointState 10.0.0.2
  HeartBeatState: generation 3, version 99
  ApplicationState "STATUS": NORMAL,5024364346313730023,16
  ApplicationState: HOST_ID: 054040af-7998-4ae4-861b-28edefe7dff,2
  ApplicationState: NET_VERSION: 9,1
  ApplicationState: RELEASE_VERSION: 3.0.0-SNAPSHOT,4
  ApplicationState: RPC_ADDRESS: 10.0.0.2,3
  ApplicationState: TOKENS: 5024364346313730023,6177183088005859152,15
  ApplicationState: LOAD: 6474848,45
  ApplicationState: DC: datacenter1,6
  ApplicationState: RACK: rack1,8
  ApplicationState: SEVERITY: 0.2557544708251953,98
EndPointState 10.0.0.3
  HeartBeatState: generation 4, version 541679
  ApplicationState "STATUS": NORMAL,-200858863971061419,16
  ApplicationState: HOST_ID: 054040af-7998-4ae4-861b-28edefe7e68d,2
  ApplicationState: NET_VERSION: 9,1
  ApplicationState: RELEASE_VERSION: 3.0.0-SNAPSHOT,4
  ApplicationState: RPC_ADDRESS: 10.0.0.3,3
  ApplicationState: TOKENS: -200858863971061419,-4098422355153952737,15
  ApplicationState: LOAD: 200000099,541675
  ApplicationState: DC: datacenter1,6
  ApplicationState: RACK: rack1,8
  ApplicationState: SEVERITY: 0.2557544708251953,541678
EndPointState 10.0.0.4
  HeartBeatState: generation 1, version 11
  ApplicationState "STATUS": NORMAL,3991949776101693449,4
  ApplicationState: HOST_ID: 054040af-7998-4ae4-861b-28edefe7e64e,2
  ApplicationState: NET_VERSION: 9,1
  ApplicationState: RELEASE_VERSION: 3.0.0-SNAPSHOT,3
  ApplicationState: RPC_ADDRESS: 10.0.0.4,3
  ApplicationState: TOKENS: 3991949776101693449,7578276156139136231,5
  ApplicationState: LOAD: 182792,6
  ApplicationState: DC: datacenter1,7
  ApplicationState: RACK: rack1,8
  ApplicationState: SEVERITY: 0.2557544708251953,9
```

In this case max version number for these endpoints are 541662, 99, 541679 and 11 respectively. A gossip digest for endpoint 10.0.0.2 would be "10.0.0.2:3:99" and essentially says "AFAIK endpoint 10.0.0.2 is running generation 3 and maximum version is 99". When the node sends GossipDigestSynMessage, there will be exactly one gossip digest per known endpoint. That is, in this case GossipDigestSynMessage contents would be: "10.0.0.1:2:541662 10.0.0.2:3:99 10.0.0.3:4:541679 10.0.0.4:1:11". HeartBeatState version number is not necessarily always the biggest, but that is the most common situation by far.

####Main code pointers:
```
Gossiper.GossipTask.run: Main gossiper loop
Gossiper.makeRandomGossipDigest: Constructs gossip digest list to be used in the GossipDigestSyn message
Gossiper.sendGossip, choose random endpoint to send GossipDigestSyn
```
###GossipDigestAckMessage
A node receiving GossipDigestSynMessage will examine it and reply with GossipDigestAckMessage, which includes two parts: gossip digest list and endpoint state list. From the gossip digest list arriving in GossipDigestSynMessage we will know for each endpoint whether the sending node has newer or older information than we do. An example to illustrate this:

Suppose that we're now in node 10.0.0.2 and our endPointState is as follows:
```
EndPointState 10.0.0.1
  HeartBeatState: generation 2, version 540001
  ApplicationState "STATUS": NORMAL,-5125166994968203647,16
  ApplicationState: HOST_ID: 054040af-7998-4ae4-861b-28edefe7e64e,2
  ApplicationState: NET_VERSION: 9,1
  ApplicationState: RELEASE_VERSION: 3.0.0-SNAPSHOT,4
  ApplicationState: RPC_ADDRESS: 10.0.0.1,3
  ApplicationState: TOKENS: -5125166994968203647,-6296082587704969101,15
  ApplicationState: LOAD: 1827, 32
  ApplicationState: DC: datacenter1,6
  ApplicationState: RACK: rack1,8
  ApplicationState: SEVERITY: 0.2557544708251953,540000
EndPointState 10.0.0.2
  HeartBeatState: generation 3, version 512
  ApplicationState "STATUS": NORMAL,5024364346313730023,16
  ApplicationState: HOST_ID: 054040af-7998-4ae4-861b-28edefe7dff,2
  ApplicationState: NET_VERSION: 9,1
  ApplicationState: RELEASE_VERSION: 3.0.0-SNAPSHOT,4
  ApplicationState: RPC_ADDRESS: 10.0.0.2,3
  ApplicationState: TOKENS: 5024364346313730023,6177183088005859152,15
  ApplicationState: LOAD: 647004848,511
  ApplicationState: DC: datacenter1,6
  ApplicationState: RACK: rack1,8
  ApplicationState: SEVERITY: 0.2557544708251953,98
EndPointState 10.0.0.3
  HeartBeatState: generation 4, version 541660
  ApplicationState "STATUS": NORMAL,-200858863971061419,16
  ApplicationState: HOST_ID: 054040af-7998-4ae4-861b-28edefe7e68d,2
  ApplicationState: NET_VERSION: 9,1
  ApplicationState: RELEASE_VERSION: 3.0.0-SNAPSHOT,4
  ApplicationState: RPC_ADDRESS: 10.0.0.3,3
  ApplicationState: TOKENS: -200858863971061419,-4098422355153952737,15
  ApplicationState: LOAD: 20099,541659
  ApplicationState: DC: datacenter1,6
  ApplicationState: RACK: rack1,8
  ApplicationState: SEVERITY: 0.2557544708251953,541650
```
Remember that the arriving gossip digest list is: "10.0.0.1:2:541662 10.0.0.2:3:99 10.0.0.3:4:541679 10.0.0.4:1:11". When the receiving end is handling this, following steps are done:

####Sort gossip digest list

Sort gossip digest list according to the difference in max version number between sender's digest and our own information in descending order. That is, handle those digests first that differ mostly in version number. Number of endpoint information that fits in one gossip message is limited. This step is to guarantee that we favor sending information about nodes where information difference is biggest (sending node has very old information compared to us).

####Examine gossip digest list
At this stage we go through the arriving gossip digest list and construct the two parts of GossipDigestAckMessage mentioned above (gossip digest list and endpoint state list). Let us go through the example digest one by one:

10.0.0.1:2:541662 In our own endPointStateMap the generation is the same, so 10.0.0.1 has not rebooted since we have last heard of it. Version number in the digest is bigger than our max version number (540001 < 541662), so we have to ask the sender what has happened since version 540001. For this purpose we include a gossip digest 10.0.0.1:2:540001, which says "I know about 10.0.0.1 only until generation 2, version 540001, please tell me anything that is newer than this".

10.0.0.2:3:99 When examining this, we notice that we know more than the sender about 10.0.0.2 (generations match, but our version is bigger 512 > 99). Sender's max version is 99, so we look for any states that are newer than this. As we can see from the endPointStateMap, there are two: ApplicationState: LOAD: 647004848,511 and and HeartBeatState(512). We send this ApplicationState to the sender. Please note that in this case we are not sending digests, as digest only tells the maximum version number. In this case we already know that there is difference, so we will send full ApplicationStates.

10.0.0.3:3:541679 In this case generations do not match. Our generation is smaller than the arriving, so 10.0.0.3 must have rebooted. We will ask all data from the sender for generation 3 starting from smallest version number 0. That is, we insert gossip digest 10.0.0.3:1259912238:0 to the reply.

10.0.0.4:1:11 We do not know anything about this endpoint, so we proceed in the same manner as 10.0.0.3 and ask for all information. Insert digest 10.0.0.4:1:0 to the reply.

At this point we have constructed GossipDigestAckMessage, which includes following information:
```
10.0.0.1:2:540001
10.0.0.3:3:0
10.0.0.4:1:0
10.0.0.2:[ ApplicationState: LOAD: 647004848,511], [ HeartBeatState: generation 3, version 512]
```
##Subscriber

##StatusChange
```java
enum ApplicationState
{
    STATUS,
    LOAD,
    SCHEMA,
    DC,
    RACK,
    RELEASE_VERSION,
    REMOVAL_COORDINATOR,
    INTERNAL_IP,
    RPC_ADDRESS,
    SEVERITY,
    NET_VERSION,
    HOST_ID,
    TOKENS
}
```
STATUS取值:
```java
 String STATUS_BOOTSTRAPPING = "BOOT";
 String STATUS_NORMAL = "NORMAL";
 String STATUS_LEAVING = "LEAVING";
 String STATUS_LEFT = "LEFT";
 String STATUS_MOVING = "MOVING";
 String REMOVING_TOKEN = "removing";
 String REMOVED_TOKEN = "removed";
 String HIBERNATE = "hibernate";
```
