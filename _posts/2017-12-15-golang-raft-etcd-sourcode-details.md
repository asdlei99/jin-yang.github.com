---
title: ETCD 源码解析
layout: post
comments: true
language: chinese
category: [program,golang,linux]
keywords: golang,go,etcd
description:
---

在上篇中介绍了 ETCD 代码中 RAFT 相关的示例代码，接着介绍与 ETCD 相关的代码。

<!-- more -->

## ETCD

### 数据结构

简单介绍下一些常见的数据结构。

#### type Storage interface

定义了存储的接口。

{% highlight go %}
type Storage interface {
    // InitialState returns the saved HardState and ConfState information.
    InitialState() (pb.HardState, pb.ConfState, error)
    // Entries returns a slice of log entries in the range [lo,hi).
    // MaxSize limits the total size of the log entries returned, but
    // Entries returns at least one entry if any.
    Entries(lo, hi, maxSize uint64) ([]pb.Entry, error)
    // Term returns the term of entry i, which must be in the range
    // [FirstIndex()-1, LastIndex()]. The term of the entry before
    // FirstIndex is retained for matching purposes even though the
    // rest of that entry may not be available.
    Term(i uint64) (uint64, error)
    // LastIndex returns the index of the last entry in the log.
    LastIndex() (uint64, error)
    // FirstIndex returns the index of the first log entry that is
    // possibly available via Entries (older entries have been incorporated
    // into the latest Snapshot; if storage only contains the dummy entry the
    // first log entry is not available).
    FirstIndex() (uint64, error)
    // Snapshot returns the most recent snapshot.
    // If snapshot is temporarily unavailable, it should return ErrSnapshotTemporarilyUnavailable,
    // so raft state machine could know that Storage needs some time to prepare
    // snapshot and call Snapshot later.
    Snapshot() (pb.Snapshot, error)
}
{% endhighlight %}

其中官方提供的 [Github Raft Example](https://github.com/coreos/etcd/tree/master/contrib/raftexample) 中使用的是库自带 MemoryStorage 。

#### type Ready struct

对于这种 IO 网络密集型的应用，提高吞吐最好的手段就是批量操作，ETCD 与之相关的核心抽象就是 Ready 结构体。

{% highlight go %}
// Ready encapsulates the entries and messages that are ready to read,
// be saved to stable storage, committed or sent to other peers.
// All fields in Ready are read-only.
type Ready struct {
    // The current volatile state of a Node.
    // SoftState will be nil if there is no update.
    // It is not required to consume or store SoftState.
    *SoftState

    // The current state of a Node to be saved to stable storage BEFORE
    // Messages are sent.
    // HardState will be equal to empty state if there is no update.
    pb.HardState

    // ReadStates can be used for node to serve linearizable read requests locally
    // when its applied index is greater than the index in ReadState.
    // Note that the readState will be returned when raft receives msgReadIndex.
    // The returned is only valid for the request that requested to read.
    ReadStates []ReadState

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

    // MustSync indicates whether the HardState and Entries must be synchronously
    // written to disk or if an asynchronous write is permissible.
    MustSync bool
}
{% endhighlight %}

#### type node struct

在 `raft/node.go` 中定义了 `type node struct` 对应的结构，一个 RAFT 结构通过 Node 表示各结点信息，该结构体内定义了各个管道，用于同步信息，下面会逐一遇到。

{% highlight go %}
type node struct {
    propc      chan pb.Message
    recvc      chan pb.Message
    confc      chan pb.ConfChange
    confstatec chan pb.ConfState
    readyc     chan Ready
    advancec   chan struct{}
    tickc      chan struct{}
    done       chan struct{}
    stop       chan struct{}
    status     chan chan Status
}
{% endhighlight %}

其实现，就是通过这些管道在 RAFT 实现与外部应用之间来传递各种消息。




#### type raft struct

在 `raft/raft.go` 中定义了 `type raft struct` 结构，其中有两个关键函数指针 `tick` 和 `step`，在不同的状态时会调用不同的函数，例如 Follower 中使用 `tickElection()` 和 `stepFollower()` 。

{% highlight go %}
type raft struct {
	id uint64

	Term uint64
	Vote uint64

	readStates []ReadState

	// the log
	raftLog *raftLog

	maxInflight int
	maxMsgSize  uint64
	prs         map[uint64]*Progress

	state StateType

	votes map[uint64]bool

	msgs []pb.Message

	// the leader id
	lead uint64
	// leadTransferee is id of the leader transfer target when its value is not zero.
	// Follow the procedure defined in raft thesis 3.10.
	leadTransferee uint64
	// New configuration is ignored if there exists unapplied configuration.
	pendingConf bool

	readOnly *readOnly

	// number of ticks since it reached last electionTimeout when it is leader
	// or candidate.
	// number of ticks since it reached last electionTimeout or received a
	// valid message from current leader when it is a follower.
	electionElapsed int

	// number of ticks since it reached last heartbeatTimeout.
	// only leader keeps heartbeatElapsed.
	heartbeatElapsed int

	checkQuorum bool
	preVote     bool

	heartbeatTimeout int
	electionTimeout  int
	// randomizedElectionTimeout is a random number between
	// [electiontimeout, 2 * electiontimeout - 1]. It gets reset
	// when raft changes its state to follower or candidate.
	randomizedElectionTimeout int

	tick func()          // 两个重要的函数指针
	step stepFunc

	logger Logger
}
{% endhighlight %}

Node 代表了 etcd 中一个节点，是 RAFT 协议核心部分实现的代码，而在 EtcdServer 的应用层与之对应的是 raftNode ，两者一对一，raftNode 中有匿名嵌入了 node 。

### 整体框架

这里的采用的是异步状态机，基于 GoLang 的 Channel 机制，RAFT 状态机作为一个 Background Thread/Routine 运行，会通过 Channel 接收上层传来的消息，状态机处理完成之后，再通过 Ready() 接口返回给上层。

其中 `type Ready struct` 结构体封装了一批更新操作，包括了：

* `pb.HardState` 需要在发送消息前持久化的消息，包含当前节点见过的最大的 term，在这个 term 给谁投过票，已经当前节点知道的 commit index；
* `Messages`  需要广播给所有 peers 的消息；
* `CommittedEntries` 已经提交但是还没有apply到状态机的日志；
* `Snapshot` 需要持久化的快照。

库的使用者从 `type node struct` 结构体提供的 ready channel 中不断 pop 出一个个 Ready 进行处理，库使用者通过如下方法拿到 Ready channel 。

{% highlight go %}
func (n *node) Ready() <-chan Ready { return n.readyc }
{% endhighlight %}

应用需要对 Ready 的处理包括:

1. 将 HardState、Entries、Snapshot 持久化到 storage；
1. 将 Messages 非阻塞的广播给其他 peers；
1. 将 CommittedEntries (已经提交但是还没有应用的日志) 应用到状态机；
1. 如果发现 CommittedEntries 中有成员变更类型的 entry，则调用 node 的 `ApplyConfChange()` 方法让 node 知道；
1. 调用 `Node.Advance()` 告诉 raft node 这批状态更新处理完，状态已经演进了，可以给我下一批 Ready 让我处理了。

注意，上述的第 4 部分和 RAFT 论文中的内容有所区别，论文中只要节点收到了成员变更日志就应用，而这里实际需要等到日志提交之后才会应用。

### 启动流程

ETCD 服务器是通过 EtcdServer 结构抽象，对应了 `etcdserver/server.go` 中的代码，包含属性 `r raftNode`，代表 RAFT 集群中的一个节点，启动入口在 `etcdmain/main.go` 文件中。

<!--
在服务器启动过程中，会调用raftNode(etcdserver/raft.go)的start方法：
-->

{% highlight text %}
main()                             etcdmain/main.go
 |-checkSupportArch()
 |-startEtcdOrProxyV2()
   |-newConfig()
   |-setupLogging()
   |-startEtcd()
   | |-embed.StartEtcd()           embed/etcd.go
   |   |-startPeerListeners()
   |   |-startClientListeners()
   |   |-etcdserver.ServerConfig() 生成新的配置
   |   |-etcdserver.NewServer()    正式启动RAFT服务etcdserver/server.go
   |-notifySystemd()
   |-select()                      等待stopped
   |-osutil.Exit()
{% endhighlight %}

这里基本上是大致的启动流程，主要是解析参数，设置日志，启动监听端口等，接下来就是其核心部分 `etcdserver.NewServer()` 。

### 启动RAFT

应用通过 `raft.StartNode()` 来启动 raft 中的一个副本，函数内部会通过启动一个 goroutine 运行。

{% highlight text %}
NewServer()                           通过配置创建一个新的EtcdServer对象etcdserver/server.go
 |-store.New()
 |-wal.Exist()
 |                                    <====会根据不同的启动场景执行相关任务
 |-startNode()                        新建一个节点，前提是没有WAL日志，且是新配置结点 etcdserver/raft.go
 | |-raft.NewMemoryStorage()
 | |-raft.StartNode()                 启动一个节点raft/node.go，开始node的处理过程<<<start>>>
 |   |-newRaft()                      创建RAFT对象raft/raft.go
 |   |-raft.becomeFollower()          这里会对关键对象初始化以及赋值，包括step=stepFollower r.tick=r.tickElection函数
 |   | |-raft.reset()                 开始启动时设置term为1
 |   |   |-raft.resetRandomizedElectionTimeout() 更新选举的随机超时时间
 |   |-newNode()                      新建节点
 |   |-node.run()                     raft/node.go 节点运行，会启动一个协程运行 <<<long running>>>
 |     |-newReady()                   新建type Ready对象
 |     |-raft.tick()                  等待n.tickc管道，这里实际就是在上面赋值的tickElection()函数
 |
 |-time.NewTicker()                   在通过&EtcdServer{}创建时新建tick时钟 etcdserver/server.go
{% endhighlight %}

启动的后台程序如下。

{% highlight go %}
func (n *node) run(r *raft) {
	var propc chan pb.Message
	var readyc chan Ready
	var advancec chan struct{}
	var prevLastUnstablei, prevLastUnstablet uint64
	var havePrevLastUnstablei bool
	var prevSnapi uint64
	var rd Ready

	lead := None
	prevSoftSt := r.softState()
	prevHardSt := emptyState

	for {
		if advancec != nil {
			readyc = nil
		} else {
			rd = newReady(r, prevSoftSt, prevHardSt)
			if rd.containsUpdates() {
				readyc = n.readyc
			} else {
				readyc = nil
			}
		}

		if lead != r.lead {
			if r.hasLeader() {
				if lead == None {
					r.logger.Infof("raft.node: %x elected leader %x at term %d",
						r.id, r.lead, r.Term)
				} else {
					r.logger.Infof("raft.node: %x changed leader from %x to %x at term %d",
						r.id, lead, r.lead, r.Term)
				}
				propc = n.propc
			} else {
				r.logger.Infof("raft.node: %x lost leader %x at term %d", r.id, lead, r.Term)
				propc = nil
			}
			lead = r.lead
		}

		select {
		// TODO: maybe buffer the config propose if there exists one (the way
		// described in raft dissertation)
		// Currently it is dropped in Step silently.
		case m := <-propc:
			m.From = r.id
			r.Step(m)
		case m := <-n.recvc:
			// filter out response message from unknown From.
			if _, ok := r.prs[m.From]; ok || !IsResponseMsg(m.Type) {
				r.Step(m) // raft never returns an error
			}
		case cc := <-n.confc:
			if cc.NodeID == None {
				r.resetPendingConf()
				select {
				case n.confstatec <- pb.ConfState{Nodes: r.nodes()}:
				case <-n.done:
				}
				break
			}
			switch cc.Type {
			case pb.ConfChangeAddNode:
				r.addNode(cc.NodeID)
			case pb.ConfChangeRemoveNode:
				// block incoming proposal when local node is
				// removed
				if cc.NodeID == r.id {
					propc = nil
				}
				r.removeNode(cc.NodeID)
			case pb.ConfChangeUpdateNode:
				r.resetPendingConf()
			default:
				panic("unexpected conf type")
			}
			select {
			case n.confstatec <- pb.ConfState{Nodes: r.nodes()}:
			case <-n.done:
			}
		case <-n.tickc:
			r.tick()
		case readyc <- rd:
			if rd.SoftState != nil {
				prevSoftSt = rd.SoftState
			}
			if len(rd.Entries) > 0 {
				prevLastUnstablei = rd.Entries[len(rd.Entries)-1].Index
				prevLastUnstablet = rd.Entries[len(rd.Entries)-1].Term
				havePrevLastUnstablei = true
			}
			if !IsEmptyHardState(rd.HardState) {
				prevHardSt = rd.HardState
			}
			if !IsEmptySnap(rd.Snapshot) {
				prevSnapi = rd.Snapshot.Metadata.Index
			}

			r.msgs = nil
			r.readStates = nil
			advancec = n.advancec
		case <-advancec:
			if prevHardSt.Commit != 0 {
				r.raftLog.appliedTo(prevHardSt.Commit)
			}
			if havePrevLastUnstablei {
				r.raftLog.stableTo(prevLastUnstablei, prevLastUnstablet)
				havePrevLastUnstablei = false
			}
			r.raftLog.stableSnapTo(prevSnapi)
			advancec = nil
		case c := <-n.status:
			c <- getStatus(r)
		case <-n.stop:
			close(n.done)
			return
		}
	}
}
{% endhighlight %}







### 消息发送

一般在 `raft/raft.go` 文件中，会通过 `r.send()` 发送，也就是 `raft.send()` 发送消息时，例如，如下是处于 Follower 状态时的处理函数 `stepFollower()` 。

{% highlight go %}
func stepFollower(r *raft, m pb.Message) {
	switch m.Type {
	case pb.MsgProp:
		if r.lead == None {
			r.logger.Infof("%x no leader at term %d; dropping proposal", r.id, r.Term)
			return
		} else if r.disableProposalForwarding {
			r.logger.Infof("%x not forwarding to leader %x at term %d; dropping proposal",
				r.id, r.lead, r.Term)
			return
		}
		m.To = r.lead
		r.send(m)
	// ... ...
	}
}
{% endhighlight %}

在同一个文件中，最终会调用 `append(r.msgs, m)`，那么这个消息是在什么时候消费的呢？

在 `type node struct` 结构体中，存在一个 readyc 的管道。

{% highlight go %}
type node struct {
	readyc chan Ready
}
{% endhighlight %}

在 `raft/node.go` 中存在一个 `node.run()` 函数，会读取所有的消息，然后同时通过管道发送。<!-- ？？？如果没有消息更新，这里会阻塞吗？？？？-->

{% highlight go %}
func newReady(r *raft, prevSoftSt *SoftState, prevHardSt pb.HardState) Ready {
	rd := Ready{
	    Entries:          r.raftLog.unstableEntries(),
		CommittedEntries: r.raftLog.nextEnts(),
		Messages:         r.msgs,
	}
	... ...
}
{% endhighlight %}

也就是说，在 `node.go` 里 `node.run()` 中构建了 Ready 对象，对象里就包涵被赋值的 msgs，并最终写到 `node.readyc` 这个管道里，如下是对应这个 case 的实现：

{% highlight go %}
case readyc <- rd:
	r.msgs = nil
	r.readStates = nil
	advancec = n.advancec
{% endhighlight %}

这里的 msgs 已经读取过并写入到了管道中，直接设置为空，并会赋值 advancec，在 `etcdserver/raft.go` 的 `raftNode.start()` 中，会起一个单独的协程读取数据；其中读取的函数实现在 raft/node.go 中：

{% highlight go %}
func (n *node) Ready() <-chan Ready { return n.readyc }
{% endhighlight %}

应用层 (也就是etcd) 读取到的 Ready 里面包含了 Vote 消息，会调用网络层发送消息出去，并且调用 Advance() 。

<!--
### 消息接收处理

其它 node 接收到网络层消息后，会调用 raft.Step() 函数。

raft.Step()  raft/raft.go
 |-raft.becomeFollower() 如果本地的term小于消息的term，就把自己置为follower
 |-raft.send() 当接收到的消息term一致时，就返回voteRespMsg为其投票

voteRespMsg 的返回信息被之前的发送方接收到了之后，就会计算收到的选票数目是否大于所有 node 的一半，如果大于则自己成为 leader，否则将自己置为 follower；

stepCandidate() raft/raft.go
 |-[case myVoteRespType] 如果收到了投票返回的消息
   |-raft.poll() 检查是否满足多数派原则


case myVoteRespType:
    gr := r.poll(m.From, m.Type, !m.Reject)
    switch r.quorum() {
    case gr:
        if r.state == StatePreCandidate {
            r.campaign(campaignElection)
        } else {
            r.becomeLeader()
            r.bcastAppend()
        }
    case len(r.votes) - gr:
        r.becomeFollower(r.Term, None)
    }
在成为leader之后，和上面的两个角色一样的，最重要的是step被置为了stepLeader，具体stepLeader中涉及到的一些操作，更多的是下一个问题会用到，这里就不多说了。

func (r *raft) becomeLeader() {
    r.step = stepLeader
}






###　提交数据

在接收到添加数据的请求时，需要先提交至 RAFT 组件在集群之间对本次提交达成一致。实现时，是将这次请求通过 golang 的 channel 提交给应用的 raftNode 结构：

func (s *kvstore) Propose(k string, v string) {
    var buf bytes.Buffer
    if err := gob.NewEncoder(&buf).Encode(kv{k, v}); err != nil {
        log.Fatal(err)
    }
	s.proposeC <- buf.String()
}

kvstore.proposeC 便是应用初始化时创建的与 raftNode 结构之间进行请求传输的 channel，对于更新请求，通过该管道将数据提交给了 raftNode 。

在 raftNode.serveChannels() 函数中，实际上会单独起一个协程处理该请求，最终阻塞到 raftNode.raft.Node.Propose() 函数中，直到完成提交后给客户端返回请求。

### RAFT指令处理

客户端的请求发送到 RAFT 组件后，最终还是通过 Ready() 暴露给应用，由应用负责写日志，发送给集群 Follower 节点等。

同样是在 raftNode.serveChannels() 中实现，其中相关代码如下：

func (rc *raftNode) serveChannels() {
    // event loop on raft state machine updates
    for {
        select {
            // 1. 通过Ready()获取到RAFT的核心指令，获取所有已经完成日志写入的请求
        case rd := <-rc.node.Ready():
            // 2. 写WAL日志
            rc.wal.Save(rd.HardState, rd.Entries)
            if !raft.IsEmptySnap(rd.Snapshot) {
                rc.saveSnap(rd.Snapshot)
                rc.raftStorage.ApplySnapshot(rd.Snapshot)
                rc.publishSnapshot(rd.Snapshot)
            }
            // 3. 这是干什么?
            rc.raftStorage.Append(rd.Entries)
            // 4. 发送给某个Follower
            rc.transport.Send(rd.Messages)
            // 5. 将已经commit的日志提交到应用状态机
            ok := rc.publishEntries(rc.entriesToApply(rd.CommittedEntries))
            if !ok {
                rc.stop()
                return
            }
            rc.maybeTriggerSnapshot()
			// 6. 完成状态机应用后，通知底层raft组件状态机应用完毕，RAFT组件可以准备下一次Ready消息了
            rc.node.Advance()
        }
    }
}
-->

在 `raft/node.go->run()` 函数中，是一个节点 (Node) 的主要处理过程，开始处于 Follower 状态，然后随着 `case <-n.tickc` 进行，开始进入选举。

<!--
## 代码走读
### 数据结构


在 campaign() 中的实现选举逻辑时，实际上实现了两个阶段 PreElection 和 Election 。？？？？

## 1. Leader选举

EtcdServer.Start()
 |-EtcdServer.start()
   |-EtcdServer.run()
     |-raftNode.start() 启动新的协程处理请求
       |-raftNode.Tick()  对于r.ticker.C管道的时钟处理
       |- 针对r.Ready()的请求

raftNode.start()

开始启动的时候，首先会设置为 Follower 状态，而且 term 为 1 。

当 node 初始化完成之后，通过 `node.run()` 开始运行，这里会启动单独的协程读取如上的管道。



raft.tickElection() 时钟处理函数，默认时间间隔为500ms raft/raft.go
 |-raft.Step() 如果超时，则开始发起选举，构造消息发送给自己，消息类型为MsgHup
   |-raft.campaign() 开始进入选举
     |-raft.becomeCandidate() 进入选举后状态由Follower转换为Candidate
	 | |-指针函数修改step=stepCandidate tick=tickElection
	 | |-raft.reset() 重置，同时term会增加1
     |-raft.send() 向每个节点发送voteMsg消息

-->

## Progress

RAFT 实现的内部，本身还维护了一个子状态机。

{% highlight text %}
                            +--------------------------------------------------------+
                            |                  send snapshot                         |
                            |                                                        |
                  +---------+----------+                                  +----------v---------+
              +--->       probe        |                                  |      snapshot      |
              |   |  max inflight = 1  <----------------------------------+  max inflight = 0  |
              |   +---------+----------+                                  +--------------------+
              |             |            1. snapshot success
              |             |               (next=snapshot.index + 1)
              |             |            2. snapshot failure
              |             |               (no change)
              |             |            3. receives msgAppResp(rej=false&&index>lastsnap.index)
              |             |               (match=m.index,next=match+1)
receives msgAppResp(rej=true)
(next=match+1)|             |
              |             |
              |             |
              |             |   receives msgAppResp(rej=false&&index>match)
              |             |   (match=m.index,next=match+1)
              |             |
              |             |
              |             |
              |   +---------v----------+
              |   |     replicate      |
              +---+  max inflight = n  |
                  +--------------------+
{% endhighlight %}

详细可以查看 [raft/design.md](https://github.com/coreos/etcd/blob/master/raft/design.md) 中的介绍，对于 Progress 实际上就是 Leader 维护的各个 Follower 的状态信息，总共分为三种状态：probe, replicate, snapshot 。

应该是 AppendEntries 接口的一种实现方式，为每个节点维护两个 Index 信息：A) matchIndex 已知服务器的最新 Index，如果还未确定则是 0；<!-- 作用是啥??????；--> B) nextIndex 用来标示需要从那个索引开始复制。那么 Leader 实际上就是将 nextIndex 到最新的日志复制到 Follower 节点。

<!--
如果多数派已经收到了请求，并进行了相关的处理，在响应之前主崩溃，那么新的日志可能在集群写入成功吗???????
-->



## 参考

两种不同的实现方式 [Github CoreOS-etcd](https://github.com/coreos/etcd)、[Github Hashicorp-raft](https://github.com/hashicorp/raft) 。

<!--
据说一个性能比lmdb好很多的存储引擎
https://github.com/leo-yuriev/libmdbx


RAFT论文的中文翻译
http://www.opscoder.info/ectd-raft-library.html
http://vlambda.com/wz_xberuk7dlD.html
http://chenneal.github.io/2017/03/16/phxpaxos%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E4%B9%8B%E4%B8%80%EF%BC%9A%E8%B5%B0%E9%A9%AC%E8%A7%82%E8%8A%B1/
https://www.jianshu.com/p/ae462a2d49a8
https://www.jianshu.com/p/ae1031906ef4
http://blog.neverchanje.com/2017/01/30/etcd_raft_core/
http://www.cnblogs.com/foxmailed/p/7173137.html
https://www.jianshu.com/p/5aed73b288f7

https://zhuanlan.zhihu.com/p/27767675
-->


{% highlight text %}
{% endhighlight %}
