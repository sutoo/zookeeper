



## ZookeeperServer
类ZookeeperServer表示一个standalone的基本的zk服务，提供共同的基本功能。
简单看它在系统中的定位：从网络IO层拿到packet，调用 `ZookeeperServer.processPacket`来处理来自client的命令，注：这个client也可以是zk集群内的其他节点。
这个网络IO层，被抽象为一个类：`ServerCnxn`
``` java
 * Interface to a Server connection - represents a connection from a client
 * to the server.
```

但是每个zk服务进程都是有一个角色，所以这个基本类被扩展了：LeaderZookeeperServer 和 FollowerZookeeperServer


## LeaderZookeeperServer
