#Node Boostrap

###执行Gossip Shadow Round
向seeds同步发送空GossipDigestSyn，得到其他节点的Gossip 信息，检查集群中其他节点的状态，节点的状态不能是STATUS_BOOTSTRAPPING，STATUS_LEAVING，STATUS_MOVING

###生成bootstrapTokens
如果cassandra.yaml没有配置initial_token, 使用Murmur3Partitioner随机生成tokens, 即bootstrapTokens
```java
//随机生成tokens代码
public static Collection<Token> getRandomTokens(TokenMetadata metadata, int numTokens)
{
    Set<Token> tokens = new HashSet<Token>(numTokens);
    while (tokens.size() < numTokens)
    {
        Token token = StorageService.getPartitioner().getRandomToken();
        if (metadata.getEndpoint(token) == null)
            tokens.add(token);
    }
    return tokens;
}
```
###BootStrapper.boostrap
计算得到stream data的数据源地址
```java
//来自RangeStreamer.getAllRangesWithStrictSourcesFor的代码片段
//当前Active ranges
TokenMetadata metadataClone = metadata.cloneOnlyTokenMap();
Multimap<Range<Token>,InetAddress> addressRanges = strat.getRangeAddresses(metadataClone);

//加入新node之后 pending ranges
metadataClone.updateNormalTokens(tokens, address);
Multimap<Range<Token>,InetAddress> pendingRangeAddresses = strat.getRangeAddresses(metadataClone);

//新节点的ranges desired ranges
desiredRanges = strat.getAddressRanges(metadataClone).get(address);

//Collects the source that will have its range moved to the new node
Multimap<Range<Token>, InetAddress> rangeSources = ArrayListMultimap.create();
for (Range<Token> desiredRange : desiredRanges)
{
    for (Map.Entry<Range<Token>, Collection<InetAddress>> preEntry : addressRanges.asMap().entrySet())
    {
        if (preEntry.getKey().contains(desiredRange))
        {
            Set<InetAddress> oldEndpoints = Sets.newHashSet(preEntry.getValue());
            Set<InetAddress> newEndpoints = Sets.newHashSet(pendingRangeAddresses.get(desiredRange));
            //Due to CASSANDRA-5953 we can have a higher RF then we have endpoints.
            //So we need to be careful to only be strict when endpoints == RF
            if (oldEndpoints.size() == strat.getReplicationFactor())
            {
                //oldEndpoints中剩余节点有且只有一个，其数据将会转移到新节点
                oldEndpoints.removeAll(newEndpoints);
                assert oldEndpoints.size() == 1 : "Expected 1 endpoint but found " + oldEndpoints.size();
            }
            rangeSources.put(desiredRange, oldEndpoints.iterator().next());
        }
    }
}
```
# Node Decommission
Decommission由nodetool执行，它从集群中删除一个节点.
###StorageService.decommission执行流程
####　计算TokenMetadata.pendingRanges
  ```java
　private void startLeaving()
　{
      //将Gossiper的status设置成LEAVING
　    Gossiper.instance.addLocalApplicationState(ApplicationState.STATUS, valueFactory.leaving(getLocalTokens()));
　    //将当前节点加入leavingEndpoints
      tokenMetadata.addLeavingEndpoint(FBUtilities.getBroadcastAddress());
      //异步更新TokenMetadata.pendingRanges
      PendingRangeCalculatorService.instance.update();
　}
  ```
#### 对每一个非SystemKeyspace, 计算出集群中的节点来负责接收将被移除节点的数据
```java
for (String keyspaceName : Schema.instance.getNonSystemKeyspaces())
{
    Multimap<Range<Token>, InetAddress> rangesMM = getChangedRangesForLeaving(keyspaceName, FBUtilities.getBroadcastAddress());
    rangesToStream.put(keyspaceName, rangesMM);
}
```
#### 数据迁移
1. 同步执行replaying batch log, 其可能产生hints
2. 同步执行streamRanges
3. 同步执行streamHints

```java
Future<?> batchlogReplay = BatchlogManager.instance.startBatchlogReplay();
Future<StreamState> streamSuccess = streamRanges(rangesToStream);
try{
    // Wait for batch log to complete before streaming hints.
    batchlogReplay.get();
}catch (ExecutionException | InterruptedException e){
    throw new RuntimeException(e);
}
Future<StreamState> hintsSuccess = streamHints();
try{
    streamSuccess.get();
    hintsSuccess.get();
}catch (ExecutionException | InterruptedException e){
    throw new RuntimeException(e);
}
```
####清除状态
```java
 private void leaveRing()
 {
     SystemKeyspace.setBootstrapState(SystemKeyspace.BootstrapState.NEEDS_BOOTSTRAP);
     tokenMetadata.removeEndpoint(FBUtilities.getBroadcastAddress());
     PendingRangeCalculatorService.instance.update();
     Gossiper.instance.addLocalApplicationState(ApplicationState.STATUS, valueFactory.left(getLocalTokens(),Gossiper.computeExpireTime()));
     int delay = Math.max(RING_DELAY, Gossiper.intervalInMillis * 2);
     Uninterruptibles.sleepUninterruptibly(delay, TimeUnit.MILLISECONDS);
}
Runnable finishLeaving = new Runnable()
{
    public void run()
    {
        shutdownClientServers();
        Gossiper.instance.stop();
        MessagingService.instance().shutdown();
        StageManager.shutdownNow();
        setMode(Mode.DECOMMISSIONED, true);
    }
};
```



