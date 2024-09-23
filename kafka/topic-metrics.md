This article is a short overview of what partition/topic metrics are exposed by a Kafka broker.

## Partition-level metrics

The constants are declared in
<https://github.com/apache/kafka/blob/3.8.0/core/src/main/scala/kafka/log/UnifiedLog.scala#L2329>

**LogStartOffset** \
`kafka.log:type=Log,name=LogStartOffset,topic=TOPIC,partition=NUMBER` \
Start offset of a partition (the earliest offset the consumers can seek to).

**LogEndOffset** \
`kafka.log:type=Log,name=LogEndOffset,topic=TOPIC,partition=NUMBER` \
End offset of a partition (the latest offset the consumers can seek to).

**Size** \
`kafka.log:type=Log,name=Size,topic=TOPIC,partition=NUMBER` \
Size (in bytes) of a partition.

**NumLogSegments** \
`kafka.log:type=Log,name=NumLogSegments,topic=TOPIC,partition=NUMBER` \
Number of log segments for a partition (controlled by topic's / broker's configuration).

## Topic-level metrics

The following need to be aggregated per broker in the cluster as they apply to all partitions.

The constants are declared in
<https://github.com/apache/kafka/blob/3.8.0/core/src/main/scala/kafka/server/KafkaRequestHandler.scala#L503>

**BytesInPerSec** \
`kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec,topic=TOPIC` \
Data (records) append rate for a given topic.
This metric does not measure replication traffic. \
<https://github.com/apache/kafka/blob/3.8.0/core/src/main/scala/kafka/server/ReplicaManager.scala#L1405-L1409>

**MessagesInPerSec** \
`kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec,topic=TOPIC` \
Record append rate for a given topic.
This metric does not measure replication traffic. \
<https://github.com/apache/kafka/blob/3.8.0/core/src/main/scala/kafka/server/ReplicaManager.scala#L1405-L1409> \
<https://github.com/apache/kafka/blob/3.8.0/storage/src/main/java/org/apache/kafka/storage/internals/log/LogAppendInfo.java#L222-L227>

**TotalProduceRequestsPerSec** \
`kafka.server:type=BrokerTopicMetrics,name=TotalProduceRequestsPerSec,topic=TOPIC` \
Produce request receive rate (a single request can reference multiple partitions, and carry a record batch for each of them - easily achieved by setting higher linger values in producers).
This metric does not measure replication traffic (as replication is achieved by Fetch requests).

**FailedProduceRequestsPerSec** \
`kafka.server:type=BrokerTopicMetrics,name=FailedProduceRequestsPerSec,topic=TOPIC` \
Produce requests that failed to be handled due to extraordinary reasons (so not things like partition not being hosted on a broker or record batch too large). \
<https://github.com/apache/kafka/blob/3.8.0/core/src/main/scala/kafka/server/ReplicaManager.scala#L1417>

**BytesOutPerSec** \
`kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec,topic=TOPIC` \
The rate of data being sent in Fetch responses (effectively data consumed by consumers).
This metric does not measure replication traffic.
<https://github.com/apache/kafka/blob/3.8.0/core/src/main/scala/kafka/server/KafkaRequestHandler.scala#L609>

**TotalFetchRequestsPerSec** \
`kafka.server:type=BrokerTopicMetrics,name=TotalFetchRequestsPerSec,topic=TOPIC` \
Fetch request receive rate.
This metric includes replication traffic (as it is implemented by the Fetch requests between brokers).

**FailedFetchRequestsPerSec** \
`kafka.server:type=BrokerTopicMetrics,name=FailedFetchRequestsPerSec,topic=TOPIC` \
Fetch requests that failed to be handled due to extraordinary reasons (so not things like partition not being hosted on a broker or replica not being available). \
<https://github.com/apache/kafka/blob/3.8.0/core/src/main/scala/kafka/server/ReplicaManager.scala#L1690>

# Broker-level metrics

**ReplicationBytesOutPerSec** \
`kafka.server:type=BrokerTopicMetrics,name=ReplicationBytesOutPerSec` \
Replication bytes out are not tracked on per-topic basis (not very surprising).
<https://github.com/apache/kafka/blob/3.8.0/core/src/main/scala/kafka/server/KafkaRequestHandler.scala#L609>

# Multitenancy

In multitenanted environments we can end up with a designs where topics are created for each of the tenants,
to maintain a high level of fairness.

An unfortunate side effect of this decision is an explosion in metric counts - we have 4 partition metrics and 7 topic metrics.
The number of data points will also depend on replication factor - as replicas have their own metrics.

## Limiting (leaders)

One of the ideas is to prevent reporting of metrics coming from replicas -
as long as the replication process works, ultimately every partition should be reporting the same offset data.

The limiting factor for this solution is whether reading from replicas is enabled -
a broker could be hosting only replicas, but if replica-consuming traffic is enabled, the topic metrics would be still meaningful. \
[KIP-392](https://cwiki.apache.org/confluence/display/KAFKA/KIP-392%3A+Allow+consumers+to+fetch+from+closest+replica)

## Limiting (top-K)

If the tenants can be consistently said to be larger (load-wise), then only a subset of all tenants could be monitored.

Which tenants are to be reported can be decided either at the compute time (e.g. returning the N largest tenants)
or be predefined (if some kind of external knowledge is available).

## Aggregation

The idea here is to reduce the number of metrics by grouping metrics together for multiple tenants / usecases.
E.g. if the topics follow the pattern of `$topic.$tenant`, then rate metrics for `$topic` could have their sum / average computed etc.

The same could be also done for tenants - all rates across all topics could be aggregated as well.

To provide higher granularity, the grouping could be done into buckets -
e.g. large vs small tenants, or simple hashing into N buckets if no external knowledge is present.
