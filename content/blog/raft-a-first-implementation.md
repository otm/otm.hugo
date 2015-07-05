+++
date = "2015-05-30T15:28:10+02:00"
draft = false
title = "Raft: A First Implementation"
type = "post"
series = ["Raft 101"]

+++

***tl;dr In this part we will implement a very simplistic cluster node using the etcd raft implementation. And with the node implementation we will start up a small cluster.***

The complete source to this post can be found at http://github.com/otm/raft-part-1

### Creating a Basic Node
Clusters are built by nodes, so lets start of by defining a node. Initially the node only needs to keep track of the raft and the persistent storage. Below is a minimal implementation of the node type and its constructor function.

~~~ go
  const hb = 1

  type node struct {
    // the raft that the cluster node will use
    // this includes the WAL
  	raft      raft.Node

    // pstore is a fake implementation of a persistent storage
    // that will be used side-by-side with the WAL in the raft
    pstore    map[string]string
  }

  func newNode(id int, peers []raft.Peer) *node {
    n := &node{
      id: id,
      cfg: &raft.Config{
    		ID:              uint64(id),
    		ElectionTick:    10 * hb,
        HeartbeatTick:   hb,
    		Storage:         raft.NewMemoryStorage(),
    		MaxSizePerMsg:   math.MaxUint16,
    		MaxInflightMsgs: 256,
    	},
      pstore: make(map[string]string),
    }

    n.raft = raft.StartNode(n.cfg, peers)

  	return n
  }

~~~

With the node comes some responsibilities:

1. Read from `Node.Ready()` channel and process the updates.
2. All persisted log entries must be made available via an implementation of the Storage interface. This can be solved by using the provided MemoryStorage type, if it is repopulated upon restart; or by implementing a disked backed implementation. In this example we will not implement this part.
3. Call `Node.Step()` when receiving a message from another node.
4. Call `Node.Tick()` at regular intervals. Internally the raft time is represented by an abstract *tick*, that controls two important timeouts, the heartbeat and election timeout.

So the state machine main loop will look like this:

~~~ go
func (n *node) run() {
	n.ticker = time.Tick(time.Second)
	for {
		select {
		case <-n.ticker:
      // Point (4) above
			n.raft.Tick()
		case rd := <-n.raft.Ready():
			// Point (2) above
		case <-n.done:
			return
		}
	}
}
~~~

### Handling Raft Ready Events
Above a large part of the control loop is left out, that is the processing of updates from the Node.Ready() channel. When receiving a message there are four important tasks:

1. Write `HardState`, `Entries` and `Snapshot` to persistent storage. **Note!** When writing to storage it is important to check the Entry Index (i). If previously persisted entries with Index >= i exist, those entries needs to be discarded. For instance this can happen if we get a cluster split with the leader in the minority part; because then the cluster can advance in the other part.  

2. Send all messages to the nodes named in the `To` field. **Note!** It is important to not send any messages until:
  * the latest HardState has been persisted
  * all Entries from previous batch have been persisted (messages from the current batch can be sent and persisted in parallel)
  * call `Node.ReportSnapshot()` if any message has type `MsgSnap` and the snapshot has been sent

3. Apply `Snapshot` and `CommitedEntries` to the state machine. If any committed Entry has the type `EntryConfChange` call `Node.ApplyConfChange` to actually apply it to the node. The configuration change can be canceled at this point by setting the `NodeId` field to zero before calling `ApplyConfChange`. Either way, `ApplyConfChange` must be called; and the decision to cancel must be based solely on the state machine and not on external information, for instance observed health state of a node.

4. Call Node.Advance() to signal readiness for the next batch. This can be done any time after step 1 is finished. **Note!** All updates must be processed in the order they were received by `Node.Ready()`

The four tasks above can be done in parallel as long as all notices above are fulfilled. In the example below `rd` is of the type `raft.Ready`

~~~ go
n.saveToStorage(rd.HardState, rd.Entries, rd.Snapshot) // (1)
n.send(rd.Messages) // (2)
if !raft.IsEmptySnap(rd.Snapshot) {
  n.processSnapshot(rd.Snapshot)  //(3)
}
for _, entry := range rd.CommittedEntries {
  n.process(entry) // (3)
  if entry.Type == raftpb.EntryConfChange {
    var cc raftpb.ConfChange
    cc.Unmarshal(entry.Data)
    n.node.ApplyConfChange(cc) // (3)
  }
}
n.raft.Advance() // (4)
~~~

Below is the `raft.Ready` type, please note that `pb` is actually `raftpb`.

~~~ go
type Ready struct {
    // The current volatile state of a Node.
    // SoftState will be nil if there is no update.
    // It is not required to consume or store SoftState
    // It is useful for logging and debugging
    *SoftState

    // The current state of a Node to be saved to stable storage BEFORE
    // Messages are sent.
    // HardState will be equal to empty state if there is no update.
    pb.HardState

    // Entries specifies entries to be saved to stable storage BEFORE
    // Messages are sent.
    Entries []pb.Entry

    // Snapshot specifies the snapshot to be saved to stable storage.
    Snapshot pb.Snapshot

    // CommittedEntries specifies entries to be committed to a
    // store/state-machine. These have previously been committed to stable
    // store.
    CommittedEntries []pb.Entry

    // Messages specifies outbound messages to be sent AFTER Entries are
    // committed to stable storage.
    // If it contains a MsgSnap message, the application MUST report back to raft
    // when the snapshot has been received or has failed by calling ReportSnapshot.
    Messages []pb.Message
}
~~~

#### Save To Storage
Saving to storage is not so advanced in a simplistic example like this:

1. Append the entries - this is basically the message to be committed
2. Set the hard state - this will commit the message
3. Apply the snapshot - this overwrites the storage with the given snapshot

~~~ go
func (n *node) saveToStorage(hardState raftpb.HardState, entries []raftpb.Entry, snapshot raftpb.Snapshot) {
	n.store.Append(entries)

	if !raft.IsEmptyHardState(hardState) {
		n.store.SetHardState(hardState)
	}

	if !raft.IsEmptySnap(snapshot) {
		n.store.ApplySnapshot(snapshot)
	}
}
~~~

#### Send
For now the RPC will be simulated with a global map which holds the nodes. With the send function there is also a matching receive function. Note, the `receive()` function calls `raft.Step()`, and that is crucial to advance the state machine.

~~~ go
func (n *node) send(messages []raftpb.Message) {
	for _, m := range messages {
    // Inspect the message (just for fun)
		log.Println(raft.DescribeMessage(m, nil))

		// send message to other node
		nodes[m.To].receive(n.ctx, m)
	}
}

func (n *node) receive(ctx context.Context, message raftpb.Message) {
	n.raft.Step(ctx, message)
}
~~~

#### Process Raft Entry
If `entry.Type` of the `raftpb.Entry` is `raftpb.EntryNormal` the message should be processed. The message will be encoded in `entry.Data`. The protocol below is very simple, and is a string on the form *key:value*

~~~ go
func (n *node) process(entry raftpb.Entry) {

	if entry.Type == raftpb.EntryNormal && entry.Data != nil {
		log.Println("normal message:", string(entry.Data))

    parts := bytes.SplitN(entry.Data, []byte(":"), 2)
		n.pstore[string(parts[0])] = string(parts[1])
	}
}
~~~

#### Process Snapshot
For now `processSnapshot` will only be a dummy implementation. But basically it should only overwrite the the current persistent storage.

~~~ go
func (n *node) processSnapshot(snapshot raftpb.Snapshot) {
	log.Printf("Applying snapshot on %v is not implemenetd yet")
}
~~~

#### Advance the Raft
At this point the only thing left to do is calling `Node.Advance()`, this signals that the node is ready for the next batch. Please note that *all* updates must be processed in the order they were received from `Node.Ready`. In our example it looks like this:

~~~ go
n.raft.Advance()
~~~

### Using the Raft
Finally it is time to start up a couple of nodes and connect them to a cluster. Below the nodes are started with predefined peers. Lastly `Node.Campaign` is called to start an election campaign; it is not necessary but it saves some time, as we do not need to wait for the election timeout. **Note!** At this point we do not have a resilient cluster, because if a node is lost there is no way to reach a majority consensus.

~~~ go
nodes[1] = newNode(1, []raft.Peer{{ID: 1}, {ID: 2}})
go nodes[1].run()

nodes[2] = newNode(2, []raft.Peer{{ID: 1}, {ID: 2}})
go nodes[2].run()

nodes[1].raft.Campaign(nodes[1].ctx)
~~~

However, all nodes in the cluster can not be known in advance. Below a node is created and added to a running cluster. It is created without any peers and a configuration change is proposed to the cluster. When the configuration change has been committed, the cluster is fault tolerant.

~~~ go
nodes[3] = newNode(3, []raft.Peer{})
go nodes[3].run()
nodes[2].raft.ProposeConfChange(nodes[2].ctx, raftpb.ConfChange{
  ID:      3,
  Type:    raftpb.ConfChangeAddNode,
  NodeID:  3,
  Context: []byte(""),
})
~~~

Next up is writing data to the cluster, that is also done by proposing a change to the cluster. Each of the writes are done to different nodes in the cluster.

~~~ go
nodes[1].node.Propose(nodes[1].ctx, []byte("key1:value1"))
nodes[2].node.Propose(nodes[2].ctx, []byte("key2:value2"))
nodes[3].node.Propose(nodes[3].ctx, []byte("key3:value3"))
~~~

Dumping the data on the nodes reveals if they have been synchronized properly.

~~~ go
for i, node := range nodes {
  fmt.Printf("** Node %v **\n", i)
  for k, v := range node.pstore {
    fmt.Printf("%v = %v", k, v)
  }
  fmt.Printf("*************\n")
}
~~~


If a node is added to the cluster, data will be replicated to the newly added node by replaying the log.
