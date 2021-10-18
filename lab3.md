[toc]
# Lab3 Fault-tolerant Key/Value Service

## 实验说明

***

在lab2的基础上，实现一个高可用KV存储服务，包括对客户端和服务端的实现。

## 实现思路

***

当时看完该lab的说明后，脑海中想到了以下几个问题：

1. KV键值在服务端如何存储起来。
2. 如何处理客户端请求，保证线性一致性。因为客户端可能会重试操作，但该操作可能已经在服务端应用过了。
3. 服务端收到客户端的操作请求后，会调用raft接口向从节点同步操作，这里的同步是需要一定时间的。go语言中有什么好的方法能在操作同步成功或超时失败后通知客户端。

对于第一个问题，参考了一些其他人的资料后，发现自己想多了，这里其实只需一个简单的map来存储KV键值，不需实现复杂的存储引擎。

对于第二个问题，raft论文在第8节给出了答案：为每个客户端每个操作都分配一个独一无二的序列号，服务端跟踪每个客户端最后一次操作的序列号，用于去取重复请求。

对于第三个问题，也是参考了一些其他人的资料，发现都是为每个raft同步操作生成一个channel来跟踪完成情况，最后我也采取了这种方式。

另外，整体框架可参考lab2的官方流程图，服务端运行着一套raft代码，客户端请求服务端进行Get/Put操作时，服务端调用raft接口同步操作，然后等待raft上报同步结果，根据结果操作存放KV的map，最后返回客户端操作结果。

### Client

客户端的实现比较简单。注意两点：一是为每个客户端每个操作生成一个序列号。二是当请求服务端超时或请求的服务端不为主节点时，能尝试连接其他服务端。

```go
func MakeClerk(servers []*labrpc.ClientEnd) *Clerk {
	ck := new(Clerk)
	ck.servers = servers
	// You'll have to add code here.
  // clientId + seqId 组成了序列号
	ck.clientId = nrand()
	ck.seqId = 0
  // 记录主节点的服务器编号
	ck.leaderServer = 0 
	return ck
}

// Get操作（Put操作类似，就不展示了）
func (ck *Clerk) Get(key string) string {

	// You will have to modify this function.
	args := GetArgs{key, ck.clientId, atomic.AddInt64(&ck.seqId, 1)}

	for {
		reply := GetReply{}
		leaderServer := ck.leaderServer
		DPrintf("Client[%v] SeqId[%v] leaderServer: %v GetArgs: %s", ck.clientId, ck.seqId, leaderServer, args.String())
		ok := ck.servers[leaderServer].Call("KVServer.Get", &args, &reply)

		if ok && (reply.Err == OK || reply.Err == ErrNoKey) {
			DPrintf("Client[%v] SeqId[%v] leaderServer: %v GetReply: Key[%v] %s", ck.clientId, ck.seqId, leaderServer, args.Key, reply.String())
			return reply.Value
		} else {
			//commit超时，或者wrongleader，需要尝试连接其他kvServer
			ck.leaderServer = (ck.leaderServer + 1) % len(ck.servers)
		}
		time.Sleep(retryTimeout)
	}
	return ""
}
```

### Server

Server的实现是重点，之前自己实现了一版，后面看了清华大佬的[实现笔记](https://github.com/LebronAl/MIT6.824-2021/blob/master/docs/lab3.md)，发现其结构很清晰明了，吸收了一点经验，重构了一把。下面讲讲自己的理解。

首先是Server的结构体，从上述分析可知，至少需要包含一个存储kv的map，一个能记录客户端最后一次操作序号的map和一个能记录每个raft同步操作结果的map，所以结构体如下：

```go
type KVServer struct {
	mu      sync.Mutex
	me      int
	rf      *raft.Raft
	applyCh chan raft.ApplyMsg
	dead    int32 // set by Kill()

	maxraftstate int // snapshot if log grows this big
	persister    *raft.Persister
	// Your definitions here.
  kvStore     map[string]string            // (key, value)
  seqMap      map[int64]int64              // (clientId, 最后的seqId)
  indexMap    map[IndexAndTerm]chan CommandResponse // (commitIndex+term, chan)
	lastApplied int
}

type IndexAndTerm struct {
	Index int
	Term  int
}
```

服务端处理客户端发来的Get请求和PutAppend请求。这里有几点需要注意：

1. 对于每次Get的读请求，这里都会生成一条raft日志去同步，简单粗暴即可保证线性一致性，这样的方式叫LogRead，虽然简单，但开销比较大。由于lab说明了不必对读请求作优化，所以就采用这样简单的方式了。对读请求的优化有ReadIndex和LeaseRead两种方法，具体可参考下这篇[文章](https://pingcap.com/zh/blog/linearizability-and-raft)。
2. 对于每次PutAppend请求，先判断是否为重复请求，再生成raft日志去同步操作，减少raft同步次数。当然在raft同步完成后，也是需要判断该请求是否为重复请求。
3. 对于重复请求，我这里是直接返回成功，其实应该返回上次的应用结果，清华大佬的[笔记](https://github.com/LebronAl/MIT6.824-2021/blob/master/docs/lab3.md)有相关说明。
4. **重点**：这里为每个需raft日志同步的请求生成一个channel来获取raft同步结果，获取这个channel的索引是index+term。若只根据index来拿，会出现很多意想不到的问题，如以为该请求同步成功了，其实是另外的请求同步成功了，直接造成请求的丢失。

```go
func (kv *KVServer) Get(args *GetArgs, reply *GetReply) {
	cmd := Op{args.Key, "", "Get", args.ClientId, args.SeqId}
	reply.Err, reply.Value = kv.clientRequestProcessHandler(cmd)
}

func (kv *KVServer) PutAppend(args *PutAppendArgs, reply *PutAppendReply) {
	cmd := Op{args.Key, args.Value, args.Op, args.ClientId, args.SeqId}
	reply.Err, _ = kv.clientRequestProcessHandler(cmd)
}

func (kv *KVServer) clientRequestProcessHandler(cmd Op) (Err, string) {
	kv.mu.Lock()
	if cmd.OpType != "Get" && kv.isDupliceRequestL(&cmd) {
		kv.mu.Unlock()
		return OK, ""
	}
	kv.mu.Unlock()

	index, term, isLeader := kv.rf.Start(cmd)
	if !isLeader {
		return ErrWrongLeader, ""
	}
	
	it := IndexAndTerm{index, term}
	kv.mu.Lock()
	ch := kv.getIndexChanL(it)
	kv.mu.Unlock()
  
	defer func() {
		kv.mu.Lock()
		delete(kv.indexMap, it)
		kv.mu.Unlock()
	}()

	select {
	case response := <-ch:
		return response.Err, response.Value
	case <-time.After(replyTimeout):
		return ErrWrongLeader, ""
	}
}
```

服务端处理raft同步结果，这里也有几点需要注意：

1. raft同步完成后，也需要判断请求是否为重复请求。因为同一请求可能由于重试会被同步多次。
2. 当要通过channel返回操作结果时，需判断当前节点为主才返回操作结果，否则返回WrongLeader。

```go
func (kv *KVServer) applyOp() {
	for !kv.killed() {
		for m := range kv.applyCh {
			if m.SnapshotValid {
				if kv.rf.CondInstallSnapshot(m.SnapshotTerm,
					m.SnapshotIndex, m.Snapshot) {
					DPrintf("kvserver: %v installsnapshot: %d", kv.me, m.SnapshotIndex)
					kv.mu.Lock()
					if m.SnapshotIndex > kv.lastApplied {
						kv.installSnapshotL(m.Snapshot)
						kv.lastApplied = m.SnapshotIndex
					}
					kv.mu.Unlock()
				}
			} else if m.CommandValid {
				kv.mu.Lock()
				if m.CommandIndex <= kv.lastApplied {
					kv.mu.Unlock()
					continue
				}

				op := m.Command.(Op)
				kv.lastApplied = m.CommandIndex

				var response CommandResponse
				if op.OpType != "Get" && kv.isDupliceRequestL(&op) {
					response = CommandResponse{OK, ""}
				} else {
					kv.seqMap[op.ClientId] = op.SeqId
					switch op.OpType {
					case "Put":
						kv.kvStore[op.Key] = op.Value
						response = CommandResponse{OK, ""}
					case "Append":
						kv.kvStore[op.Key] += op.Value
						response = CommandResponse{OK, ""}
					case "Get":
						if value, ok := kv.kvStore[op.Key]; ok {
							response = CommandResponse{OK, value}
						} else {
							response = CommandResponse{ErrNoKey, ""}
						}
					}
				}
        
				if currentTerm, isLeader := kv.rf.GetState(); isLeader {
					DPrintf("ChanRespone Command:%v Response:%v commitIndex:%v currentTerm: %v", op, response, m.CommandIndex, currentTerm)
					ch := kv.getIndexChanL(IndexAndTerm{m.CommandIndex, currentTerm})
					ch <- response
				}

				if kv.maxraftstate != -1 && kv.persister.RaftStateSize() > kv.maxraftstate {
					kv.SnapshotL(m.CommandIndex)
				}
        
				kv.mu.Unlock()
			}
		}

	}
}
```

最后是日志压缩，实现比较简单。将服务端的一些重要变量编码后生成snapshot，然后通知raft进行日志压缩。

```go
// 保存快照
func (kv *KVServer) SnapshotL(snapshotIndex int) {
	DPrintf("Snapshot: server: %v snapshotIndex:%v", kv.me, snapshotIndex)
	w := new(bytes.Buffer)
	e := labgob.NewEncoder(w)
	e.Encode(kv.kvStore)
	e.Encode(kv.seqMap)
	e.Encode(kv.lastApplied)
	data := w.Bytes()
	kv.rf.Snapshot(snapshotIndex, data)
}

// 安装快照
func (kv *KVServer) installSnapshotL(snapshot []byte) {
	r := bytes.NewBuffer(snapshot)
	d := labgob.NewDecoder(r)
	var kvStore map[string]string
	var seqMap map[int64]int64
	var lastApplied int

	if d.Decode(&kvStore) != nil || d.Decode(&seqMap) != nil ||
		d.Decode(&lastApplied) != nil {
		DPrintf("error to read the snapshot data")
	} else {
		kv.kvStore = kvStore
		kv.seqMap = seqMap
		kv.lastApplied = lastApplied
	}
}
```

## 总结

***

* 要注意上述的一些造成请求丢失的坑点，考虑清楚返回请求成功的情况。
* 该lab我也是并行测试了几千次（测了一周多）是没问题，但不保证真的没有bug，欢迎指教！要测试那么多次是因为想连同raft的bug一并测试了。By the way，测试这几个lab时，看每次测试结果的心情都有点像之前搞深度学习训练时看loss下降和accuracy的心情，真就有那味了。
* 该lab利用raft构建了一个高可用的KV存储服务，是对raft的一个应用。完成了该lab后，加深了对raft的理解，特别是如何和上层应用进行交互。
* 由于源码不能公开，上述只放了部分核心代码。如需源码可以联系我，欢迎讨论。存储小白，也请多指教。