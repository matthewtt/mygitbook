# Cassandra Cache
## Cache初始化
Cache由三种类型，KeyCache，RowCache，CounterCache, 默认KeyCache开启, 类结构图
![ICache 类图](images/CacheClassStructure.png)

Cache接口和Map接口类似, 增加capacity显示容量, 在put和remove时不返回Value, 避免在SerializingCache中deserialize在offheap的Value数据．

![ICache 层级关系](images/CacheClassHierarchy.png)

###KeyCache 和 CounterCache 初始化
KeyCache 和 CounterCache的代码类似，此只列出KeyCache
```java
private AutoSavingCache<KeyCacheKey, RowIndexEntry> initKeyCache()
{
    long keyCacheInMemoryCapacity = DatabaseDescriptor.getKeyCacheSizeInMB() * 1024 * 1024;
    ICache<KeyCacheKey, RowIndexEntry> kc = ConcurrentLinkedHashCache.create(keyCacheInMemoryCapacity);
    AutoSavingCache<KeyCacheKey, RowIndexEntry> keyCache = new AutoSavingCache<>(
                                                           kc,
                                                           CacheType.KEY_CACHE,
                                                           new KeyCacheSerializer());
    int keyCacheKeysToSave = DatabaseDescriptor.getKeyCacheKeysToSave();
    keyCache.scheduleSaving(DatabaseDescriptor.getKeyCacheSavePeriod(), keyCacheKeysToSave);
    return keyCache;
}
```
ConcurrentLinkedHashCache的创建
```java
public static ConcurrentLinkedHashCache<K, V> create(long weightedCapacity)
{
    EntryWeigher<K, V> entryWeigher = new EntryWeigher<K, V>()
    {
        public int weightOf(K key, V value)
        {
            long size = key.unsharedHeapSize() + value.unsharedHeapSize();
            return (int) size;
        }
    };
    ConcurrentLinkedHashMap<K, V> map = new ConcurrentLinkedHashMap.Builder<K, V>()
                                        .weigher(entryWeigher)
                                        .maximumWeightedCapacity(weightedCapacity)
                                        .concurrencyLevel(DEFAULT_CONCURENCY_LEVEL)
                                        .build();
    return new ConcurrentLinkedHashCache<>(map);
}
```

###RowCache 初始化
RowCache初始化唯一的不同就是Cache的实现换成SerializingCache
```java
ICache<RowCacheKey, IRowCacheEntry> rc = new SerializingCacheProvider().create(rowCacheInMemoryCapacity);

public ICache<RowCacheKey, IRowCacheEntry> create(long capacity)
{
    return SerializingCache.create(capacity, new RowCacheSerializer());
}

public static <K, V> SerializingCache<K, V> create(long weightedCapacity, ISerializer<V> serializer)
{
    Weigher<RefCountedMemory> weigher = new Weigher<RefCountedMemory>()
    {
        public int weightOf(RefCountedMemory value)
        {
            long size = value.size();
            return (int) size;
        }
    };
    return new SerializingCache<>(weightedCapacity, weigher, serializer);
}

private SerializingCache(long capacity, Weigher<RefCountedMemory> weigher, ISerializer<V> serializer)
{
    this.serializer = serializer;
    EvictionListener<K,RefCountedMemory> listener = new EvictionListener<K, RefCountedMemory>()
    {
        public void onEviction(K k, RefCountedMemory mem)
        {
            mem.unreference();
        }
    };
    this.map = new ConcurrentLinkedHashMap.Builder<K, RefCountedMemory>()
               .weigher(weigher)
               .maximumWeightedCapacity(capacity)
               .concurrencyLevel(DEFAULT_CONCURENCY_LEVEL)
               .listener(listener)
               .build();
}
```

#Cache使用
###put操作
![put 调用关系](images/CachePutStack.png)
可以看到除了三个缓存类型都会执行装载文件中的缓存数据，Counter和Key都会执行插入操作, RowCache没有执行插入操作，仅仅执行replace操作。

###putIfAbsent, replace
只在RowCache中使用

###get
有两种形式，不走统计的使用 InstrumentingCache.getInternal，走统计的使用InstrumentingCache.get
```java
//InstrumentingCache.get代码
public V get(K key)
{
    V v = map.get(key);
    metrics.requests.mark();
    if (v != null)
        metrics.hits.mark();
    return v;
}
```
![get 调用关系](images/CacheGetStack.png)

//todo 详细解释

###remove
![remove 调用关系](images/CacheRemoveStack.png)









