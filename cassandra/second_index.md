# Second Index
second index:
Cassandra's built-in secondary indexes are best on a column family having many rows that contain the indexed value. The more unique values that exist in a particular column, the more overhead you will have, on average, to query and maintain the index.
倾向于查询许多值都相同的column, 而非在column family中值唯一的column
An advantage of secondary indexes is the operational ease of populating and maintaining the index. Secondary indexes are built in the background automatically, without blocking reads or writes.

缺点：
Do not use secondary indexes to query a huge volume of records for a small number of results．If want to query such case, need to maintain a dynamic column family as a form of an index instead of using a secondary index. For columns containing unique data, it is sometimes fine performance-wise to use secondary indexes for convenience, as long as the query volume to the indexed column family is moderate and not under constant load.

维护：
To perform a hot rebuild of a secondary index, use the nodetool utility rebuild_index command.
