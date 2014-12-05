# Cassandra Cache
cache由三种类型，KeyCache，RowCache，CounterCache, 默认只进行KeyCache. 在初始化ColumnFamilyStore时，会load已经save的KeyCache
```java
//ColumnFamilyStore
if (caching.keyCache.isEnabled())
    CacheService.instance.keyCache.loadSaved(this);
