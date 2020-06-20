# 介绍 zookeeper 的持久化相关

## 大概的做法
transaction 被写入 txnlog 中，并定期进行 snapshot。

## 基本概念
### ZKDatabase
```java
/**
 * This class maintains the in memory database of zookeeper
 * server states that includes the sessions, datatree and the
 * committed logs. It is booted up  after reading the logs
 * and snapshots from the disk.
 */
```
这个类代表了 zk 在内存中的所有数据，包括 datatree， sessions 和 committed logs。

* `DataTree dataTree`，即内存中的 data数据。
* `sessionsWithTimeouts`，即 session 数据。

### FileTxnSnapLog
这个类为 txnlog and snapshot 的一个帮助类，里面提供了一些基本操作
通过成员变量可以看到此类整合了2部分功能，即txnlog 和 snapshot

```java
    //the direcotry containing the
    //the transaction logs
    private final File dataDir;
    //the directory containing the
    //the snapshot directory
    private final File snapDir;
    private TxnLog txnLog;
    private SnapShot snapLog;
```
主要支持的方法：
* fastForwardFromEdits 回放txnlog中的记录，修改 datatree 与 sessions。不读snapshot。txnlog 是以zxid结尾的大量文件，这个回放方法会选择DataTree中
  的 lastProcessedZxid+1 来选择读取的文件。
* restore 从 snapshot 中 restore，而后再执行 fastForwardFromEdits

## transaction logs (TxnLogs)

### append
在`SyncRequestProcessor`中，最重要的一件事是把 request 同步到磁盘。
```java
  // track the number of records written to the log
                    if (zks.getZKDatabase().append(si)) {
```
最终调用的为上面介绍的帮助类`FileTxnSnapLog`的 append 方法。
```java
    public boolean append(Request si) throws IOException {
        return txnLog.append(si.getHdr(), si.getTxn());
    }
```
最终将 record 写入文件。

### roll
txnlog 的文件后缀为当前的 zxid ，所以这个文件到一定大小的时候，就会被外部通过调用 roll 方法，来重新生成一个新文件，旧文件还继续保留。

## SnapShot
即 zookeeper 数据的快照，默认的实现为：
```java
 class FileSnap implements SnapShot
```
方法比较简单：序列化和反序列化。主要靠外部来调用，触发 snapshot。
在`SyncRequestProcessor`中被触发，逻辑入口为：
```java
    public void takeSnapshot(){
        try {
            txnLogFactory.save(zkDb.getDataTree(), zkDb.getSessionWithTimeOuts());
        } catch (IOException e) {
            LOG.error("Severe unrecoverable error, exiting", e);
            // This is a severe error that we cannot recover from,
            // so we need to exit
            System.exit(10);
        }
    }
```
snapshot 是如何同时进行的？
如何保证先加载snapshot，再回放txnlog，能保证数据完整性呢。
首先进行snapshot之前，会把 `DataTree` 的 `lastProcessedZxid`写入到snapshot的文件名中。
然后再将 `DataTree` 逐步写入 snapshot 文件。中间容许用户继续对 `DataTree` 做更新操作。
但是可以保证的是`lastProcessedZxid`之前的操作都是在 snapshot中的。因此，在加载完 snapshot ，回放 txnLog时，也是通过 snapshot 的 
``来控制如何选择回放的日志
```java
 public long fastForwardFromEdits(DataTree dt, Map<Long, Integer> sessions,
                                     PlayBackListener listener) throws IOException {
        TxnIterator itr = txnLog.read(dt.lastProcessedZxid+1);
        long highestZxid = dt.lastProcessedZxid;
```
代码中的read的参数含义为从哪个 zxid 开始读取, 返回一个迭代器。这里的迭代器会返回所有 zxid 之后的 txnlog，他对上层屏蔽了txnlog的底层存储方式，
即分多个物理文件保存。







