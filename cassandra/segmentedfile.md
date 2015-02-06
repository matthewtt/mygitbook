# SegmentedFile
用作Index.db和Data.db的对象表示，Index.db默认使用MmappedSegmentedFile，Data.db默认使用 CompressedPoolingSegmentedFile．先说下定义SegmentedFile的原因，JVM一次做多只能map 2GB的文件，超过这个值就会出现IllegalArgumentException，对于Index.db和Data.db, 如果其大小超过这个值就会分段map.

###MmappedSegmentedFile
MmappedSegmentedFile中创建Segment代码片段
```java
MappedByteBuffer segment = size <= MAX_SEGMENT_SIZE
                           ? raf.getChannel().map(FileChannel.MapMode.READ_ONLY, start, size)
                           : null;
segments[i] = new Segment(start, segment);
```

在MmappedSegmentedFile.Builder中有一个`List<Long> boundaries`概念, 在节点初始化时，对每个Index.db的文件进行逻辑分段，其大小不超过MAX_SEGMENT_SIZE

###CompressedPoolingSegmentedFile
CompressedPoolingSegmentedFile当前只在Data.db中使用．不存在分段，但多了一个CompressionInfo.db　文件，由类 CompressionMetadata　表示，其结构如下：
```java
//从CompressionInfo得到压缩之前Data.db的大小
public final long dataLength;
//整个压缩文件Data.db的大小
public final long compressedFileLength;
public final boolean hasPostCompressionAdlerChecksums;
//所有的compressed文件的chunk offset, 存储在Off-Heap
private final Memory chunkOffsets;
//所有的compressed文件的chunk offset个数
private final long chunkOffsetsSize;
public final String indexFilePath;
//从CompressionInfo.db得到数据创建
public final CompressionParameters parameters;
```

```java
class CompressionParameters{
    //默认是LZ4Compressor
    public final ICompressor sstableCompressor;
    //默认是65536
    private final Integer chunkLength;
    //默认1.0
    private volatile double crcCheckChance;
}
```
每次执行CompressedPoolingSegmentedFile#getSegment, 都会先从FileCacheService执行一次查询，每个PoolingSegmentedFile都有一个`FileCacheService.CacheKey`, 具体代码如下：
```java
public FileDataInput getSegment(long position){
    RandomAccessReader reader = FileCacheService.instance.get(cacheKey);
    if (reader == null)
        reader = createReader(path);
    reader.seek(position);
    return reader;
}
```
在CompressedPoolingSegmentedFile#createReader将创建CompressedRandomAccessReader, 如下是关于CompressedRandomAccessReader的创建
```java
protected CompressedRandomAccessReader(String dataFilePath, CompressionMetadata metadata, PoolingSegmentedFile owner)
{
    super(new File(dataFilePath), metadata.chunkLength(), owner);
    this.metadata = metadata;
    checksum = metadata.hasPostCompressionAdlerChecksums ? new Adler32() : new CRC32();
    compressed = ByteBuffer.wrap(new byte[metadata.compressor().initialCompressedBufferLength(metadata.chunkLength())]);
}
```
CompressedRandomAccessReader有两点需要注意在Super class RandomAccessReader中创建chunkLength长度的buffer, 在LZ4Compressor同样会创建chunkLength的buffer, 在LZ4Compressor中存放的buffer是用于读取Data.db中的已被压缩的数据，RandomAccessReader中存放的是解压后的数据．



