# Cassandra 启动

## DatabaseDescritor初始化

读取cassandra.yaml配置文件, 存储到Config类， 从Config类中读取配置参数, 来初始化部分配置类

重点谈论下DatabaseDescriptor对SystemKeyspace的初始化, System Keyspace是写死在KSMetaData中
```java
static void applyConfig(Config config) throws ConfigurationException{
    List<KSMetaData> systemKeyspaces = Arrays.asList(KSMetaData.systemKeyspace());
    assert systemKeyspaces.size() == Schema.systemKeyspaceNames.size();
    for (KSMetaData ksmd : systemKeyspaces)
        //将每个KSMetaData下面的CFMetaData的UUID存入cfIdMap, KSMetaData存入keyspaces
        Schema.instance.load(ksmd);
}
```

Schema类结构

```java
NonBlockingHashMap<String, KSMetaData> keyspaces; //ksName --> KSMataData
NonBlockingHashMap<String, Keyspace> keyspaceInstances; //ksName --> Keyspace
ConcurrentBiMap<Pair<String, String>, UUID> cfIdMap; //Pair<ksName, cfName> --> cfId
```

##Load keyspaces 和 UDFs
所有的keyspaces存储在system.schema_keyspaces中,ColumnFamily都存储在system.schema_columnfamilies，UTMetaData存储在system.schema_usertypes, UDFs存储在system.schema_functions，在CassandraDaemon.setup中触发。
```java
void setup(){
    DatabaseDescriptor.loadSchemas();
    Functions.loadUDFFromSchema();
}

//DatabaseDescriptor.loadSchemas
// load kSMetaData, 同时也初始化Keyspace, 存入Schema.keyspaceInstances
public static void loadSchemas(){
    Schema.instance.load(DefsTables.loadFromKeyspace());
    Schema.instance.updateVersion();
}
```
DefsTables类中全是静态方法，在loadFromKeyspace中主要关注

* ColumnFamilyStore创建及查询，查询具体细节参考查询一章

 由于创建ColumnFamilyStore是在Keyspace类中执行，需要先初始化Keyspace, 再对每个ColumnFamily创建相应的ColumnFamilyStore

 Keyspace类结构
 ```java
 //包含当前节点的token/identifier. token将会在节点之间gossip传递, 还维持其他节点负载信息柱状统计图
 KSMetaData metadata;
 OpOrder writeOrder;
 ConcurrentHashMap<UUID, ColumnFamilyStore> columnFamilyStores; //cfId --> ColumnFamilyStore
 AbstractReplicationStrategy replicationStrategy;
 ```
 初始化
 ```java
 private Keyspace(String keyspaceName, boolean loadSSTables){
        metadata = Schema.instance.getKSMetaData(keyspaceName);//从schema取出KSMetaData
        createReplicationStrategy(metadata);//创建ReplicationStrategy
        this.metric = new KeyspaceMetrics(this);
        for (CFMetaData cfm : new ArrayList<CFMetaData>(metadata.cfMetaData().values()))
        {
            initCf(cfm.cfId, cfm.cfName, loadSSTables);
        }
  }
  //在initCF中创建ColumnFamilyStore
  columnFamilyStores.putIfAbsent(cfId, ColumnFamilyStore.createColumnFamilyStore(this, cfName, loadSSTables));
  //在Keyspace.open中对RowCache初始化
  for (ColumnFamilyStore cfs : keyspaceInstance.getColumnFamilyStores())
      cfs.initRowCache();
  ```
* 对结果的组装

```java
public static Collection<KSMetaData> loadFromKeyspace(){
    //执行查询
    List<Row> serializedSchema = SystemKeyspace.serializedSchema(SystemKeyspace.SCHEMA_KEYSPACES_CF);
    List<KSMetaData> keyspaces = new ArrayList<>(serializedSchema.size());
    for (Row row : serializedSchema){
        if (Schema.invalidSchemaRow(row) || Schema.ignoredSchemaRow(row))
            continue;
        //还需要对ColumnFamily, UserType进行查询
        keyspaces.add(KSMetaData.fromSchema(row, serializedColumnFamilies(row.key), serializedUserTypes(row.key)));
    }
    return keyspaces;
}
//在KSMetaData中对Row执行反序列化
public static KSMetaData fromSchema(Row serializedKs, Row serializedCFs, Row serializedUserTypes){
    Map<String, CFMetaData> cfs = deserializeColumnFamilies(serializedCFs);
    UTMetaData userTypes = new UTMetaData(UTMetaData.fromSchema(serializedUserTypes));
    return fromSchema(serializedKs, cfs.values(), userTypes);
}

//KSMataData结构
String name;
Class<? extends AbstractReplicationStrategy> strategyClass;
Map<String, String> strategyOptions;
Map<String, CFMetaData> cfMetaData; // cfName --> CFMetaData
boolean durableWrites; //是否写入Commit log
UTMetaData userTypes;

//CFMetaData结构, 主要参数
UUID cfId; //内部id, 不会暴露给用户
String ksName;
String cfName;
ColumnFamilyType cfType; //standard　or super
CellNameType comparator; // bytes, long, timeuuid, utf8, etc.
```

