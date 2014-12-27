# Atomic Batches
Batch允许客户端将多个相关的updates组成一个statement, 如果部分replicas在更新batch中失败，coordinator将会自动hint这些batch. 但是当coordinator在hint这些batch时失败，将会得到部分跟新的batches．

在之前依靠客户端，通过重新发送batch到不同的coordinator来处理这种情况，因为cassandra的write是幂等的，这种方法通常已经足够．但是当客户端和coordinator同时down掉，尤其是它们在同一个datacenter时,　数据将很难恢复．一些复杂的客户端实现了client-side commitlog来处理这种情况，但很明显由服务端来处理这种情况更为恰当．

##使用atomic batches
从Cassandra1.2开始，Batches默认是atomic. 但是batches的原子性相比于非原子性有30%的性能损失．对于性能要求高的操作使用BEGIN UNLOGGED BATCH，它不会提供原子性保证．同时1.2开始为batch counter updates引入BEGIN COUNTER BATCH. 不同于其他write, couter updates不是idempotent，从batchlog自动replaying batches是不安全的．在同一个partition更新多个counters时，Counter batches 完全是为了改善性能．

###内部实现
Atomic batches使用batchlog表
```
CREATE TABLE system.batchlog (
    id uuid PRIMARY KEY,
    data blob,
    version int,
    written_at timestamp
) WITH compaction = {};
```

当写入atomic batch, 首先会序列化batch成data blob到batchlog表.　在batch中的行被成功写入时或则hinted, 删除batchlog entry.
####组装写入batchlog表
```java
static Mutation getBatchlogMutationFor(Collection<Mutation> mutations, UUID uuid, int version, long now)
{
    ColumnFamily cf = ArrayBackedSortedColumns.factory.create(SystemKeyspace.BatchlogTable);
    CFRowAdder adder = new CFRowAdder(cf, SystemKeyspace.BatchlogTable.comparator.builder().build(), now);
    adder.add("data", serializeMutations(mutations, version))
         .add("written_at", new Date(now / 1000))
         .add("version", version);
    return new Mutation(SystemKeyspace.NAME, UUIDType.instance.decompose(uuid), cf);
}

private static void syncWriteToBatchlog(Collection<Mutation> mutations, Collection<InetAddress> endpoints, UUID uuid) throws WriteTimeoutException
{
    AbstractWriteResponseHandler handler = new WriteResponseHandler(endpoints,
                                                                    Collections.<InetAddress>emptyList(),
                                                                    ConsistencyLevel.ONE,
                                                                    Keyspace.open(SystemKeyspace.NAME),
                                                                    null,
                                                                    WriteType.BATCH_LOG);
    MessageOut<Mutation> message = BatchlogManager.getBatchlogMutationFor(mutations, uuid, MessagingService.current_version)
                                                  .createMessage();
    for (InetAddress target : endpoints)
    {
        int targetVersion = MessagingService.instance().getVersion(target);
        if (target.equals(FBUtilities.getBroadcastAddress()) && OPTIMIZE_LOCAL_REQUESTS)
        {
            insertLocal(message.payload, handler);
        }
        else if (targetVersion == MessagingService.current_version)
        {
            MessagingService.instance().sendRR(message, target, handler, false);
        }
        else
        {
            MessagingService.instance().sendRR(BatchlogManager.getBatchlogMutationFor(mutations, uuid, targetVersion)
                                                              .createMessage(),
                                               target,
                                               handler,
                                               false);
        }
    }
    handle.get();
}
```
####发送batch data到replica
```java
private ReplayWriteResponseHandler sendSingleReplayMutation(final Mutation mutation, long writtenAt, int ttl)
{
    Set<InetAddress> liveEndpoints = new HashSet<>();
    String ks = mutation.getKeyspaceName();
    Token tk = StorageService.getPartitioner().getToken(mutation.key());
    for (InetAddress endpoint : Iterables.concat(StorageService.instance.getNaturalEndpoints(ks, tk),
                                                StorageService.instance.getTokenMetadata().pendingEndpointsFor(tk, ks)))
    {
        if (endpoint.equals(FBUtilities.getBroadcastAddress()))
            mutation.apply();
        else if (FailureDetector.instance.isAlive(endpoint))
            //直接发送到replica, 而非写入hinted表
            liveEndpoints.add(endpoint);
        else
            //写入hints表
            StorageProxy.writeHintForMutation(mutation, writtenAt, ttl, endpoint);
    }
    if (liveEndpoints.isEmpty())
        return null;
    ReplayWriteResponseHandler handler = new ReplayWriteResponseHandler(liveEndpoints);
    MessageOut<Mutation> message = mutation.createMessage();
    for (InetAddress endpoint : liveEndpoints)
    　　//直接发送到replica
        MessagingService.instance().sendRR(message, endpoint, handler, false);
    return handler;
}
```
###如果发送batch data 到replica失败，写入hints表
```java
private void writeHintsForUndeliveredEndpoints(int startFrom)
{
    try
    {
        // Here we deserialize mutations 2nd time from byte buffer.
        // but this is ok, because timeout on batch direct delivery is rare
        // (it can happen only several seconds until node is marked dead)
        // so trading some cpu to keep less objects
        List<Mutation> replayingMutations = replayingMutations();
        for (int i = startFrom; i < replayHandlers.size(); i++)
        {
            Mutation undeliveredMutation = replayingMutations.get(i);
            int ttl = calculateHintTTL(replayingMutations);
            ReplayWriteResponseHandler handler = replayHandlers.get(i);
            if (ttl > 0 && handler != null)
                for (InetAddress endpoint : handler.undelivered)
                    StorageProxy.writeHintForMutation(undeliveredMutation, writtenAt, ttl, endpoint);
        }
    }
    catch (IOException e)
    {
        logger.error("Cannot schedule hints for undelivered batch", e);
    }
}
```
###定时replay batches

启动schedule线程
```java
//replay task
Runnable runnable = new WrappedRunnable()
{
    public void runMayThrow() throws ExecutionException, InterruptedException
    {
        replayAllFailedBatches();
    }
};
batchlogTasks.scheduleWithFixedDelay(runnable, StorageService.RING_DELAY, REPLAY_INTERVAL, TimeUnit.MILLISECONDS);
```
replayAllFailedBatches执行情况：
1. 组装RateLimiter
2. 分页取出batchlog entry
3. 按页replay batchlog
4. force flush 内存中的batchlog tombstones, 执行compaction task清空batchlog space
