# Session in Zookeeper

## 概念
session 在概念上指 client 端与 server端的一个会话，

在 server 端， session 有两种，他们可以互相转化。
* Global session 全局session，在每个server上都存在
* local session 只在当前请求的server上存在，但只能进行读操作，要是要进行写操作，就得升级为全局session
## session 数据的持久化
todo 参考下关于zk的持久化相关
## local session 的引入
参照这个 zookeeper 的 jira ：https://issues.apache.org/jira/browse/ZOOKEEPER-1147
考虑有大量的只读 client 连接集群的时候，因为对创建 session 的处理和其他的 update 操作是一样的，所以 session 的创建/删除，很容易就把整个集群的资源耗尽。由此引出了 `local session`来应对大量连接时产生的问题。

local session的特点：
* 不能创建临时节点
* 一旦丢失了，就无法通过 session id 和 password 重建
* session info 只会存在于连接着的当前 zk server，leader 对这个 session 无感知，也不会将 session 信息写入磁盘
* session 超时也是由当前连接着的 zk server 来负责。
在接口层面引入的改动：
* client 创建连接的时候，可以指定是否为 local session
* local session 会被升级为 global session，当执行写操作的时候

sessin 升级的逻辑：
对于 follower 来说，在`FollowerRequestProcessor`中，处理request的时候，先会判断下是否需要升级 session.如果需要升级，则将 upgradeRequest 先放入 queue 等待后续转发给 leader。 这个 upgradeRequest 的类型为 `OpCode.createSessio`。
```java
    public void processRequest(Request request) {
        if (!finished) {
            // Before sending the request, check if the request requires a
            // global session and what we have is a local session. If so do
            // an upgrade.
            Request upgradeRequest = null;
            try {
                upgradeRequest = zks.checkUpgradeSession(request);
            } catch (KeeperException ke) {
                if (request.getHdr() != null) {
                    request.getHdr().setType(OpCode.error);
                    request.setTxn(new ErrorTxn(ke.code().intValue()));
                }
                request.setException(ke);
                LOG.info("Error creating upgrade request",  ke);
            } catch (IOException ie) {
                LOG.error("Unexpected error in upgrade", ie);
            }
            if (upgradeRequest != null) {
                queuedRequests.add(upgradeRequest);
            }
            queuedRequests.add(request);
        }
    }
```


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











