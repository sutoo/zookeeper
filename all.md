



## ZookeeperServer
类ZookeeperServer表示一个standalone的基本的zk服务，提供共同的基本功能。
简单看它在系统中的定位：从网络IO层拿到packet，调用 `ZookeeperServer.processPacket`来处理来自client的命令，注：这个client也可以是zk集群内的其他节点。
这个网络IO层，被抽象为一个类：
``` java
 /**
 * Interface to a Server connection - represents a connection from a client
 * to the server.
 */
public abstract class ServerCnxn implements Stats, Watcher {}
```
它具体实现类有：`NIOServerCnxn` 和 `NettyServerCnxn`

但是每个zk服务进程都是有一个角色，所以这个基本类被扩展了：LeaderZookeeperServer 和 FollowerZookeeperServer

RequestProcessor
ZookeeperServer被设计为通过内部的一个processor的链条来处理request。

## LeaderZookeeperServer
处理链条为：
LeaderRequestProcessor -> PrepRequestProcessor -> ProposalRequestProcessor -> CommitProcessor -> ToBeAppliedRequestProcessor -> FinalRequestProcessor

依次分析上面的每个Processor的作用：
* LeaderRequestProcessor 负责本地session的更新 //todo session相关
* PrepRequestProcessor 初始化与request关联的事物
* ProposalRequestProcessor todo
* CommitProcessor 用来匹配本地 submitted request和发过来的committed request
* ToBeAppliedRequestProcessor
* FinalRequestProcessor 
与
