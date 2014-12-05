# Cassandra查询过程


### service

* `StorageService` <br/>
  重要参数<br/>

  ```java
  TokenMetadata tokenMetadata; //管理 the token/endpoint metadata 信息 */
  ```

### net

* `MessagingService` <br/>
  重要参数<br/>

  ```java
  EnumMap<Verb, IVerbHandler>(Verb.class) verbHandlers; //通过verb查询messaging handlers
  SimpleCondition listenGate; //
  NonBlockingHashMap<InetAddress, OutboundTcpConnectionPool> connectionManagers;
  ArrayList<SocketThread> socketThreads;
  ExpiringMap<Integer, CallbackInfo> callbacks; //配置参数DatabaseDescriptor.getMinRpcTimeout
  ```

### locator
* `AbstractReplicationStrategy` 所有replication strategies的父类 <br/>

  ```java
  String keyspaceName;
  Keyspace keyspace;
  Map<String, String> configOptions;
  TokenMetadata tokenMetadata;
  IEndpointSnitch snitch;
  NonBlockingHashMap<Token, ArrayList<InetAddress>> cachedEndpoints;
  ```

### sstable

* `SStable`, 在SequenceFile之上创建这个类，按顺序存放数据，但排序方式取决于application. 一个单独的索引文件将被维护，包含SSTable　keys和在SSTable中的偏移位置. 在SSTable被打开时，每隔indexInterval的key会被读到内存。每个SSTable 为keys保存一份bloom filter文件.

  ```java
   Descriptor descriptor;
   Set<Component> components;
   CFMetaData metadata;
   IPartitioner partitioner;
   boolean compression;
   DecoratedKey first, last;
  ```

* `SSTableReader extends SSTable`, 通过Keyspace.onStart打开SSTableReaders; 在此之后通过SSTableWriter.renameAndOpen被创建. 不要在存在的SSTable文件上re-call　open()，使用ColumnFamilyStore保存的references来使用.

  ```java
   SegmentedFile ifile, dfile;
   IndexSummary indexSummary;
   IFilter bf;
   StatsMetadata sstableMetadata;
  ```

