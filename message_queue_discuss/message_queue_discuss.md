# 消息队列闲谈

## Kafka Controller and leader election
Each broker have a controller, and the leader controller is elected by zookeeper.

## Replica in Kafka
![image](https://pic1.zhimg.com/v2-3979c9a7498aa531fe932a4dce9fbb48_r.jpg)
## leader/follower model for replication
When some records are produced to a topic, they are [first written to the log of the leader broker](https://github.com/apache/kafka/blob/cb9557a99050bf317cb347cc7756a61f25ce032d/core/src/main/scala/kafka/server/ReplicaManager.scala#L592). The followers fetch the records from the leader, write them to their logs, and acknowledges the leader. A record is considered committed when it has been written to the log of the leader and all in-sync replicas. The leader will notify the producer once the record is committed. The producer can then safely assume that the record will not be lost as long as at least one in-sync replica remains alive. 

The leader controller elects the leader and monitors all replicas. If the leader fails, the controller will elect a new leader.

Replica State: [None](https://github.com/apache/kafka/blob/cb9557a99050bf317cb347cc7756a61f25ce032d/core/src/main/scala/kafka/server/ReplicaManager.scala#L170), [Offline](https://github.com/apache/kafka/blob/cb9557a99050bf317cb347cc7756a61f25ce032d/core/src/main/scala/kafka/server/ReplicaManager.scala#L180), and [Online](https://github.com/apache/kafka/blob/cb9557a99050bf317cb347cc7756a61f25ce032d/core/src/main/scala/kafka/server/ReplicaManager.scala#L175)

Online: The replica is in-sync with the leader and is ready to serve requests.

Offline: The replica is syncing with the leader and is not ready to serve requests.

None: The replica is invalid and is not ready to serve requests.

## Implementation of Log(Record): Skiplist
![image](https://www.singlestore.com/images/cms/blog-posts/img_blog_post_featured_what-is-skiplist-why-skiplist-index-for-memsql.png)
Now Kafka uses skiplist to implement the log. The skiplist is a data structure that allows fast search within an ordered sequence of elements.

Insert: O(log n), SkipList supports random insert, but in Kafka, the insert is always at the end of the list.

Search: O(log n)

Delete: O(log n)

See [leetcode 1206](https://leetcode.com/problems/design-skiplist/) for more details.

## Log Compression
A basic idea to reduce bandwidth and storage is to compress the log. Kafka supports various compression algorithms, including GZIP, Snappy, LZ4, and ZSTD.

The compression usually happens at the producer side, it can also happens at the broker side, and the decompression happens at the consumer side. 
```
producer -> broker -> consumer
```
I guess the compression at the broker side is for the case that the producer is not able to compress the data, e.g. IoT devices.
![image](https://note.youdao.com/yws/api/personal/file/CEF9B00C83E348DCB3351219EB7AC980?method=download&shareKey=c89b8a76cf002f96f5299988d9899e36)

## In-order delivery
Kafka guarantees in-order delivery for each partition. The producer will wait for the previous record to be committed before sending the next record. The consumer will read the records in the same order as they are written to the log.

In such case, we can only have one producer and one consumer for each partition. If we have multiple producers, the records from different producers may be out of order. If we have multiple consumers, the records from different consumers may be out of order.

## Exactly Once in Kafka
### Target use case: 
stream processing
```mermaid
graph TD;
[Kafka] --> [Stream Processor] : consume topics
[Stream Processor] --> [Kafka] : produce topics
```
Expected behavior: a record is processed exactly once even if the stream processor crashes/get timeout.

stream processor is both producer and consumer.
### Where can failure happen?
1. Broker failure
2. Producer to Broker RPC failure
3. Client failure

### Idempotence:
Assign a sequence number to each record.

If a broker receives a record with a sequence number that it has already seen, it will ignore the record. So when a broker fails, the records that have been written to the log will not be written again.

## Transection in Kafka
[blog](https://www.confluent.io/blog/transactions-apache-kafka/)

Common use case:

