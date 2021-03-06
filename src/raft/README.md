# 2A测试·选举

## 如何保证在一轮选举中，一个follower只对一个leader投票
设置标记位，投票成功标记这个follower的归属。同时还要监控这个标记位，标记位的清除时机应该是超过心跳超时的时间leader没有联系自己，那么就清除该标记位。

## follower -> candidate
如果在选举超时的时间内没有leader联系自己，那么follower成为candidate

## candidate -> leader
* 返回的请求中，大多数节点同意了候选者的投票请求
* 要满足在当前term中一个raft节点只投票一次，需要为该raft节点设置voteFor标记，代表赞成的候选者id；如果这个标记被占用，那么拒绝其他投票请求。该标记的后续的更新，是由选举出的leader完成；标记的清除是在leader心跳超时的时间内没有leader联系自己，那么raft节点自动清除。

## leader -> follower
* 发现更高的term，退回follower。这个发现更高term的时机可以是，leader发起心跳时检查返回值时；leader收到其它leader的心跳时；leader接收到候选者的选举请求时。
* 发起心跳请求时发现每个请求都超时，那么可能是网络原因，那么当前leader退回follower，这么做是为了完成leader的快速失败，因为如果是网络原因失联，那么在当前leader失败之后重连回集群后，很可能日志已经不是最新了，并且已经有了新的leader那么就能快速以follower的身份工作，减少集群中同时存在两个leader时间。

## candidate -> follower
* 发现更高的term时退回follower，这个时机可以是在发起投票的过程中；接收到别人的投票请求时。
* 接收到leader的心跳。

# 2B测试·日志复制

## 一个忽视掉的问题
在2B测试的节点失联测试中发现一个问题：


&#8192;&#8192;&#8192;&#8192;存在三个节点的情况下，有一个节点失联，集群应当正常工作，因为此时应该是满足大多数的原则。但是偶尔测试会出现失败的情况，就某个日志无法达成一致，并且leader会退回follower，正常的节点也会出现leader心跳超时的情况。

简单来说就是单个坏掉的节点，影响了leader的正常时序，而leader和follower耦合的地方在于，leader需要经常与follower保持心跳，以维护自己leader的地位。所以如果由于单个节点失联，延迟了心跳的处理流程，那么就会影响leader下一轮的心跳请求。从代码上看，也确实是这个原因。

其实关于这个问题，在当初的设计中已经考虑到了，就是对于follower的请求是有超时机制的，但是该时间设计过长，加上每次心跳的时间超过了选举超时时间，就会出现有新的candicate产生，leader看到更大的term然后退化的情况。

所以有两种如下的处理方法：
* 关于leader的超时时间，应该满足以下不等式：leader发送心跳的超时时间+leader发送心跳循环的sleep时间<选举超时时间
* 当leader的大部分心跳请求都返回的时候，那么就不再等待那个比较慢的节点了。再下一轮的心跳中，再次尝试连接。

同时再follower接收日志成功的同时，也需要更新了leader的心跳时间，因为成功的日志更新，也说明了合法的leader是存在的。


## 当一个节点正处于和master寻找匹配点的时候，如何避免向该节点发起新的日志追加请求
leader为其他每个节点维护一个标记位，如果该标记位被占用，说明已经有线程在处理了。

## log提交到状态机的时机是什么时候
应该起一个独立的线程定时检查已经提交到状态机的index和由leader更新的index的大小，并将差值提交到状态机。


## 如何处理过时的leader发起的日志追加请求
过时的leader，term必然小，所以就直接退回follower

## 如何在AppendEntries操作中对日志进行校验
* 当日志为第一次初始化时，
* 当日志不是第一次初始化时。首先要校验raft节点的term，请求的比较小就拒绝；然后是要校验请求的raft和接收的raft在相同index处的term是否相同，不匹配就返回false，然后问题是，不匹配的时候，leader和follower如何做才能使日志保持一直；如果已经存在的entry和新的entry冲突，那么就删除冲突的，以leader为准。

## 当follower和leader日志不一致时，如何使其保持一致
* leader会为每个follower维持nextIndex，如果follower当前的日志和leader冲突，那么leader就减小nextIndex的值，然后重试
* 此处有一个优化就是，当follower返回失败的时候，就返回冲突term的第一个index，这样就可以跳过很多index

## 日志复制过程中的一致性问题
首先看两段描述

>The leader
appends the command to its log as a new entry, then issues AppendEntries RPCs in parallel to each of the other
servers to replicate the entry. When the entry has been
safely replicated (as described below), the leader applies
the entry to its state machine and returns the result of that
execution to the client

>A log entry is committed once the leader
that created the entry has replicated it on a majority of
the servers (e.g., entry 7 in Figure 6)
The leader keeps track of the highest index it knows
to be committed, and it includes that index in future
AppendEntries RPCs (including heartbeats) so that the
other servers eventually find out. Once a follower learns
that a log entry is committed, it applies the entry to its
local state machine (in log order)

第一段描述讲的是leader在确认客户端提交的entry是committed状态时，会把entry提交到状态机，然后响应客户端。

第二段讲的是commited的判定方式，复制到大多数server时，leader会认为是已经提交的。并且在随后的AppendEntries操作中告知follower这件事，follower会把该entry提交到状态机。

那么问题就来了，如果leader在知道entry已经committed之后，并把这件事告诉follower之前挂掉了，同时已经告诉客户端提交成功。那么这时就会出现数据不一致的情况。客户端以为已经提交，然而事实上并没有。Raft对此的处理方式式对下次选出的leader做了限制，只有拥有最新的日志的节点才有资格成为follower，这也是复制到大多数节点时，判定提交成功的意义所在。日志新旧的比较取决于term，先比较term，term大的新；term相同则比较log的长度，log长的新。

由此也可以看出raft算法保证的是最终一致性，因为在客户端得知提交成功，到follower接收到确认提交的commitIndex之间，存在时间延迟，正常的延迟应该是leader心跳的时间间隔。当然对于客户端来说，也取决于算法的具体实现，如果可以读leader和follower，那么必然有延迟。如果只能读leader，那么保证的应该就是强一致性。

还有一种情况需要讨论，客户端提交了请求，leader进行复制操作，只有少部分机器复制了最新的entry，leader会等到当前entry committed，并且应用到状态机之后，才会告诉客户端已成功。那么在当前entry committed之前，leader挂了，那么只有少部分机器存了最新的entry，同时此时客户端应该会收到一个插入异常。问题就在于，此时用户也无法判断，当前操作到底是否提交成功，那么这种情况如何处理。假如说是持有最新entry的机器成为leader，那么其实是生效了。但如果不是那些机器，那么其实是没生效，这种情况貌似就没法处理了。

针对可能出现的重复提交的情况，客户端每次提交一个命令都可以为某个命令添加一个序列号，提交的命令重试的时候序列号应该不变，那么在下次leader选出来的时候，如果该命令已经执行，那么凭序列号校验就可以避免重复执行。但还有一个问题就是，客户端在挂掉重启的情况下，那么就真的可能存在重复执行的情况，即使是重复提交的命令有可能因为重启而生成新的序列号，然后就可能会出现重复提交的情况。