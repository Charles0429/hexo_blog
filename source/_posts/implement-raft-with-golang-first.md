---
title: golang实现Raft（一）：选主
date: 2016-11-03 10:05:28
categories: 分布式
tags:
  - golang
  - raft
  - 分布式
---

# 介绍

本文为golang实现Raft第一篇，主要描述了如何使用golang实现选主，文中的代码框架来自于MIT 6.824课程，包括rpc框架及测试用例。

# Raft选主

根据Raft论文，选主模块主要包括三大功能：

- candidate状态下的选主功能
- leader状态下的心跳广播功能
- follower状态下的确认功能

## candidate状态下的选主功能

candidate状态下的选主功能需要关注两个方面：

- 何时进入candidate状态，进行选主？
- 选主的逻辑是怎样的？

首先，来讨论何时进入candidate状态，进行选主。

在一定时间内没有收到来自leader或者其他candidate的有效RPC时，将会触发选主。这里需要关注的是有效两个字，要么是leader发的有效的心跳信息，要么是candidate发的是有效的选主信息，即server本身确认这些信息是有效的后，才会重新更新超时时间，超时时间根据raft论文中推荐设置为[150ms,300ms]，并且每次是随机生成的值。

其次，来讨论选主的逻辑。

server首先会进行选主的初始化操作，即server会增加其term，把状态改成candidate，然后选举自己为主，并把选主的RPC并行地发送给集群中其他的server，根据返回的RPC的情况的不同，做不同的处理：

- 该server被选为leader
- 其他的server选为leader
- 一段时间后，没有server被选为leader

针对情况一，该server被选为leader,当前仅当在大多数的server投票给该server时。当其被选为主时，会立马发送心跳消息给其他的server，来表明其已经是leader，防止发生新的选举。

针对情况二，其他的server被选为leader，它会收到leader发送的心跳信息，此时，该server应该转为follower，然后退出选举。

针对情况三，一段时间后，没有server被选为leader，这种情况发生在没有server获得了大多数的server的投票情况下，此时，应该发起新一轮的选举。

## leader状态下的心跳广播功能

当某个server被选为leader后，需要广播心跳信息，表明其是leader，主要在以下两个场景触发：

- server刚当选为leader
- server周期性的发送心跳消息，防止其他的server进入candidate选举状态

leader广播心跳的逻辑为，如果广播的心跳信息得到了大多数的server的确认，那么更新leader自身的选举超时时间，防止发生重新选举。

## follower状态下的确认功能

主要包括对candidate发的选举RPC以及leader发来的心跳RPC的确认功能。

对于选举RPC，假设candidate c发送选举RPC到该follower，由于follower每个term只能选举一个server，因此，只有当一个follower没有选举其他server的时候，并且选举RPC中的candidate c的term大于或等于follower的term时，才会返回选举当前candidate c为主，否则，则返回拒绝选举当前candidate c为主。

对于leader的心跳RPC，如果leader的心跳的term大于或等于follower的term，则认可该leader的心跳，否则，不认可该leader的心跳。

备注：本节所讨论的选举功能仅限于raft论文5.2，还未考虑选举过程中日志相关的信息以及选主过程中出现宕机等场景，此部分功能将在日志复制功能实现中再描述。

## 选主实现

### RPC

根据上述功能，需要以下的RPC：

- 选举RPC
- 心跳RPC

选举RPC包括的信息如下：

```go
//       
// example RequestVote RPC arguments structure.
//       
type RequestVoteArgs struct {                                                                                                                                                           
  // Your data here.
  Term         int
  CandidateId  int 
  LastLogIndex int 
  LastLogTerm  int 
} 
```
其中，Term表示当前candidate的term，而candidateId表明当前candidate的ID，全局唯一。其他两个参数将在日志复制中功能完成后再使用，暂时先不讨论。

选举RPC的回复包含的信息如下：

```go
type RequestVoteReply struct {
  // Your data here.
  Term        int
  VoteGranted bool
}  
```

其中Term表示确认的server的term，如果candidate的term小于它，将会更新其term；VoteGranted表明回复的follower是否给其投票。

心跳RPC包含的信息如下：

```go
type AppendEntriesArgs struct {
  Term     int
  LeaderId int
} 
```

其中Term为leader的term，LeaderId为当前leader的ID，全局唯一。

心跳RPC的回复信息包括：

```go
type AppendEntriesReply struct {
  Term    int
  Success bool
}  
```
包括回复的server的Term信息，以及是否认可该leader继续为主。

### candidate状态下的选主功能

根据前面描述，主要的逻辑为

- 等待选举超时
- 增加term，置状态为follower，并且选举自己为leader
- 向其他的server并行地发送选举RPC，直到碰到上述描述的三种情况退出

```go
func (rf *Raft) election() {
  var result bool
  for {
    timeout := randomRange(150, 300)
    printTime()
    fmt.Printf("candidate=%d wait election timeout=%d\n", rf.me, timeout)
    rf.setMessageTime(milliseconds())
    for rf.lastMessageTime+timeout > milliseconds() {
      select {
      case <-time.After(time.Duration(timeout) * time.Millisecond):
        printTime()
        fmt.Printf("candidate=%d, lastMessageTime=%d, timeout=%d, plus=%d, now=%d\n", rf.me, rf.lastMessageTime, timeout, rf.lastMessageTime+timeout, milliseconds())
        if rf.lastMessageTime+timeout <= milliseconds() {
          break
        } else {
          rf.setMessageTime(milliseconds())
          timeout = randomRange(150, 300)
          continue
        }
      }
    } 
      
    printTime()
    fmt.Printf("candidate=%d timeouted\n", rf.me)
    // election till success
    result = false
    for !result {
      result = rf.election_one_round()
    } 
  }   
}     

```
上述代码中，首先等待选举超时，超时后，会进入真正的选举逻辑`election_one_round`，其代码如下

```go
func (rf *Raft) becomeCandidate() {                                                                                                                                                     
  rf.state = 1   
  rf.setTerm(rf.currentTerm + 1)
  rf.votedFor = rf.me
  rf.currentLeader = -1
} 
```

首先，进入candidate状态，增加其term，然后，选举自己。

```go
   for i := 0; i < len(rf.peers); i++ {
      if i != rf.me {
        var args RequestVoteArgs
        server := i
        args.Term = rf.currentTerm
        args.CandidateId = rf.me
        var reply RequestVoteReply
        printTime()
        fmt.Printf("candidate=%d send request vote to server=%d\n", rf.me, i)
        go rf.sendRequestVoteAndTrigger(server, args, &reply, rpcTimeout)
      }
    } 
    done = 0
    triggerHeartbeat = false
    for i := 0; i < len(rf.peers)-1; i++ {
      printTime()
      fmt.Printf("candidate=%d waiting for select for i=%d\n", rf.me, i)
      select {
      case ok := <-rf.electCh:
        if ok {
          done++
          success = done >= len(rf.peers)/2 || rf.currentLeader > -1
          success = success && rf.votedFor == rf.me
          if success && !triggerHeartbeat {
            triggerHeartbeat = true
            rf.mu.Lock()
            rf.becomeLeader()
            rf.mu.Unlock()
            rf.heartbeat <- true
            printTime()
            fmt.Printf("candidate=%d becomes leader\n", rf.currentLeader)
          }
        }
      }
      printTime()
      fmt.Printf("candidate=%d complete for select for i=%d\n", rf.me, i)
    } 
```

接着，向除自己外的server发送选举RPC，等待server的回复，当成功返回数目到多数派时（包含自己在内），则宣布自己称为leader，即`becomeLeader`，如下

```go
func (rf *Raft) becomeLeader() {                                                                                                                                                        
  rf.state = 2   
  rf.currentLeader = rf.me
}            
```

即，修改自身状态为leader。然后，给发送心跳的线程发送`rf.heartbeat <-true`，通知心跳线程开始发心跳包。

最后，一轮结束之后，检测是否达到三个退出条件之一：

```go
    if (timeout+last < milliseconds()) || (done >= len(rf.peers)/2 || rf.currentLeader > -1) {
      break     
    } else {    
      select {  
      case <-time.After(time.Duration(10) * time.Millisecond):
      }         
    } 
```

即，`timeout+last < milliseconds()`达到超时时间；或者`done >= len(rf.peers)/2`，server成为leader；或者`rf.currentLeader > -1`，有其他server选为leader。

### leader状态下的广播心跳功能

首先，来看触发心跳的逻辑

```go
func (rf *Raft) sendLeaderHeartBeat() {
  timeout := 20
  for {   
    select {
    case <-rf.heartbeat:
      rf.sendAppendEntriesImpl()
    case <-time.After(time.Duration(timeout) * time.Millisecond):
      rf.sendAppendEntriesImpl()
    }     
  }       
}     
```
分为两个方面：

- 第一个为刚当选为leader后，需要马上发送心跳信息，防止新的选举发生
- 第二个是leader周期性的发送心跳信息，来宣布自己为主

真正的广播心跳的逻辑如下：

```go
func (rf *Raft) sendAppendEntriesImpl() {
  if rf.currentLeader == rf.me {
    var args AppendEntriesArgs
    var success_count int
    timeout := 20
    args.LeaderId = rf.me
    args.Term = rf.currentTerm
    printTime()  
    fmt.Printf("broadcast heartbeat start\n")
    for i := 0; i < len(rf.peers); i++ {
      if i != rf.me {
        var reply AppendEntriesReply
        printTime()
        fmt.Printf("Leader=%d send heartbeat to server=%d\n", rf.me, i)
        go rf.sendHeartBeat(i, args, &reply, timeout)
      }          
    }            
    for i := 0; i < len(rf.peers)-1; i++ {
      select {   
      case ok := <-rf.heartbeatRe:
        if ok {  
          success_count++
          if success_count >= len(rf.peers)/2 {
            rf.mu.Lock()
            rf.setMessageTime(milliseconds())
            rf.mu.Unlock()
          }      
        }        
      }          
    }            
    printTime()  
    fmt.Printf("broadcast heartbeat end\n")
    if success_count < len(rf.peers)/2 {
      rf.mu.Lock()
      rf.currentLeader = -1
      rf.mu.Unlock()
    }            
  }              
}             
```

先是向集群中所有的其他server广播心跳，分为两种结果：

- 收到了大多数server的确认，则更新leader的超时时间，防止重新进入选举状态
- 未收到大多数server的确认，则会退出发送心跳的逻辑，即置currentLeader = -1，此后，自然会有选举超时的server重新发起选举

### follower状态下的确认功能

包括对选举RPC的确认已经对心跳RPC的确认。

选举RPC的确认逻辑如下

```go
func (rf *Raft) RequestVote(args RequestVoteArgs, reply *RequestVoteReply) {
  // Your code here.
  currentTerm, _ := rf.GetState()
  if args.Term < currentTerm {
    reply.Term = currentTerm
    reply.VoteGranted = false
    printTime() 
    fmt.Printf("candidate=%d term = %d smaller than server = %d, currentTerm = %d\n", args.CandidateId, args.Term, rf.me, rf.currentTerm)
    return      
  }             
              
  if rf.votedFor != -1 && args.Term <= rf.currentTerm {
    reply.VoteGranted = false
    rf.mu.Lock()
    rf.setTerm(max(args.Term, currentTerm))
    reply.Term = rf.currentTerm
    rf.mu.Unlock()
    printTime() 
    fmt.Printf("rejected candidate=%d term = %d server = %d, currentTerm = %d, has_voted_for = %d\n", args.CandidateId, args.Term, rf.me, rf.currentTerm, rf.votedFor)
  } else {      
    rf.mu.Lock()
    rf.becomeFollower(max(args.Term, currentTerm), args.CandidateId)
    rf.mu.Unlock()
    reply.VoteGranted = true
    fmt.Printf("accepted server = %d voted_for candidate = %d\n", rf.me, args.CandidateId)
  }             
}             

```
如果当前server的term大于candidate的term，或者当前server已经选举过其他server为leader了，那么返回拒绝的RPC，否则，则返回成功的RPC，并置自身状态为follower。

心跳的RPC的逻辑如下

```go
func (rf *Raft) AppendEntries(args AppendEntriesArgs, reply *AppendEntriesReply) {
  if args.Term < rf.currentTerm {
    reply.Success = false
    reply.Term = rf.currentTerm
  } else {       
    reply.Success = true
    reply.Term = rf.currentTerm
    rf.mu.Lock() 
    rf.currentLeader = args.LeaderId
    rf.votedFor = args.LeaderId
    rf.state = 0 
    rf.setMessageTime(milliseconds())
    printTime()  
    fmt.Printf("server = %d learned that leader = %d\n", rf.me, rf.currentLeader)
    rf.mu.Unlock()
  }              
} 
```

如果follower的term大于leader的term，则返回拒绝的RPC，否则，返回成功的RPC。

本文中所有的代码都在[Raft](https://github.com/Charles0429/toys/blob/master/6.824/src/raft/raft.go)。

PS:
本博客更新会在第一时间推送到微信公众号，欢迎大家关注。

![qocde_wechat](http://oserror.com/images/qcode_wechat.jpg)

# 参考文献

- [In Search of an Understandable Consensus Algorithm(Extended Version)](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf)
