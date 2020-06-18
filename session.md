# Session in Zookeeper

## 概念
session 在概念上指 client 端与 server端的一个会话，

在 server 端， session 有两种，他们可以互相转化。
* Global session 全局session，在每个server上都存在
* local session 只在当前请求的server上存在，但只能进行读操作，要是要进行写操作，就得升级为全局session

## 服务端 session 相关
* SessionTracker
负责 session 的创建和管理


## session 的创建
考虑到 client 可以连 zk 集群中的任意一个节点，可以是 leader 也可以是 follower。
所以创建 session 的逻辑是在 ZookeeperServer 中的，逻辑为接受到一个新的连接请求便创建一个 session，即创建一个类型为`OpCode.createSessio`的`Request`。
这个请求，这个请求依次会被以下的processor处理：
* FollowerRequestProcessor
如果是 follower 接收到了创建 session 的请求，如果不是 local session 则会转给 leader 来处理。
这点和其他的对数据的写操作是类似的。
```java
                case OpCode.createSession:
                case OpCode.closeSession:
                    // Don't forward local sessions to the leader.
                    if (!request.isLocalSession()) {
                        zks.getFollower().request(request);
                    }

```

最后会被`FinalRequestProcessor`处理：
```java
            case OpCode.createSession: {
                zks.serverStats().updateLatency(request.createTime);

                lastOp = "SESS";
                cnxn.updateStatsForResponse(request.cxid, request.zxid, lastOp,
                        request.createTime, Time.currentElapsedTime());

                zks.finishSessionInit(request.cnxn, true);
                return;
            }
```

## Session 的保持活
简单描述下，服务端定期检查 session 是否过期，客户端为了让 session 保持住，不断的发送PING 来触发服务端对 session 的重新激活，也就是 触发了 `touchSession`这个动作。











