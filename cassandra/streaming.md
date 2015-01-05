# Streaming
处理节点之间的数据交换，主要用在repair, bootstrap, bulkload, move等等．接收和转移文件被划分到同一个stream session, 每个desctination创建一个stream session, 所有的stream sessions关联stream plan.

##公共接口
####StreamPlan
用来创建streaming plan(what to transfer, what to reqeust), 在其内部创建StreamSessions和其他节点交互，将所有的StreamSessions关联到StreamResultFuture，其异步返回StreamState.
####StreamResultFuture
表示StreamPlan的future结果．添加StreamEventHandler(监听StreamEvent)到StreamResultFuture, 可以用来跟踪StreamPlan的执行情况
####StreamState
Streaming的执行状态．可以得到在运行的streaming快照StreamResultFuture#getCurrentState，也可以得到最终状态StreamResultFuture#get.
####StreamManager
管理所有的streaming progress, 提供stream JMX notification
####StreamEventHandler
处理相应的event

##内部接口
####StreamSession
对每个destination, 包含所有的stream tasks(IN/OUT)
```java
Map<UUID, StreamTransferTask> transfers = new ConcurrentHashMap<>();
Map<UUID, StreamReceiveTask>  receivers = new ConcurrentHashMap<>();
```
####StreamTask
代表每个IN/OUT task, 每个task都属于一个Stream session. StreamReceiveTask 接收从destination来的SSTable, 发送complete reply. StreamTransferTask, 发送Stream request到destination, 等待reply.
####ConnectionHandler
发送/接收Stream messages

##Message Flow
####初始化
当执行StreamPlan#execute, 为每个endpoint初始化StreamSessions, 每个stream session交换messages和文件直到它完成或则失败．

如下是在StreamSession中初始化IncomingMessageHandler和OutgoingMessageHandler：
```java
public void initiate() throws IOException
{
    //创建incoming socket
    Socket incomingSocket = session.createConnection();
    //启动incloming 线程
    incoming.start(incomingSocket, StreamMessage.CURRENT_VERSION);
    //发送InitMessage
    incoming.sendInitMessage(incomingSocket, true);
    //创建outgoing Socket
    Socket outgoingSocket = session.createConnection();
    //启动outgoing 线程
    outgoing.start(outgoingSocket, StreamMessage.CURRENT_VERSION);
    //发送InitMessage
    outgoing.sendInitMessage(outgoingSocket, false);
}
```
####建立Stream连接
每个stream session 发送StreamInit message 到其他节点． StreamInit包含stream flag告知节点开始streaming 流程．一旦节点收到StreamInit, 它将创建同样的StreamPlan和StreamSession.

StreamInit结构
```java
 public final InetAddress from;
 public final int sessionIndex;
 public final UUID planId;
 public final String description;
 //outgoing or incoming
 public final boolean isForOutgoing;
 public final boolean keepSSTableLevel;
 ```
 ####准备Stream
 一旦initiator建立起connection, 它将发送Prepare message. Prepare message包含stream requests和stream summaries. Stream requests为keyspace/columnfamilies请求数据. Stream summaries用来传输数据到peer node. 它描述有多少文件和数据的大小．

一旦peer node接收到Prepare message, perr StreamSession将会准备transfer/receive文件．如果initiator请求数据, peer node 发回包含与接收到的stream requests相应的stream summaries．

StreamRequest结构
```java
public final String keyspace;
public final Collection<Range<Token>> ranges;
public final Collection<String> columnFamilies = new HashSet<>();
public final long repairedAt;
```
StreamSummary结构
```java
public final UUID cfId;
public final int files;
public final long totalSize;
```
####File Transfer
当双方完成prepare, 他们开始传输文件，每个transfer都包含file header和data.　File header用来描述data内容，包含有keyspace/columnfamily name,  sections of the file, sequence number, and if compressed, compression information. Sequence number 用来做retry.

#####文件传输细节
文件传输分为两种情况：Ranges和File传输
######Ranges传输
接收到StreamRequest, 从StreamRequest#ranges计算出要传输的数据在SSTable中的具体位置，创建SSTableStreamingSections，最后组装成OutgoingFileMessage，添加到StreamTransferTask中．
#####File传输
与Ranges传输类似，略

如下是计算Ranges在SSTable中的位置
```java
public List<Pair<Long,Long>> getPositionsForRanges(Collection<Range<Token>> ranges)
{
    // use the index to determine a minimal section for each range
    List<Pair<Long,Long>> positions = new ArrayList<>();
    for (Range<Token> range : Range.normalize(ranges))
    {
        AbstractBounds<RowPosition> keyRange = range.toRowBounds();
        RowIndexEntry idxLeft = getPosition(keyRange.left, Operator.GT);
        long left = idxLeft == null ? -1 : idxLeft.position;
        if (left == -1)
            // left is past the end of the file
            continue;
        RowIndexEntry idxRight = getPosition(keyRange.right, Operator.GT);
        long right = idxRight == null ? -1 : idxRight.position;
        if (right == -1 || Range.isWrapAround(range.left, range.right))
            // right is past the end of the file, or it wraps
            right = uncompressedLength();
        if (left == right)
            // empty range
            continue;
        positions.add(Pair.create(left, right));
    }
    return positions;
}
```
如下是在StreamTransferTask中添加Transfer file
```java
public synchronized void addTransferFile(SSTableReader sstable, long estimatedKeys, List<Pair<Long, Long>> sections, long repairedAt)
{
    assert sstable != null && cfId.equals(sstable.metadata.cfId);
    OutgoingFileMessage message = new OutgoingFileMessage(sstable, sequenceNumber.getAndIncrement(), estimatedKeys, sections, repairedAt, session.keepSSTableLevel());
    files.put(message.header.sequenceNumber, message);
    totalSize += message.header.size();
}
```

FileMessageHeader结构
```java
public final UUID cfId;
public final int sequenceNumber;
//SSTable version
public final String version;
public final SSTableFormat.Type format;
public final long estimatedKeys;
public final List<Pair<Long, Long>> sections;
public final CompressionInfo compressionInfo;
public final long repairedAt;
public final int sstableLevel;
```










