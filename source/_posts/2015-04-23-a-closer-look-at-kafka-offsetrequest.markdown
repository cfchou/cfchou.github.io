---
layout: post
title: "A Closer Look at Kafka OffsetRequest"
date: 2015-04-23 12:04:59 +0800
comments: true
categories: kafka
---

###TL;NR###
When using Kafka's simple consumer API to query offsets of messages in a given partition, the result we get are either the offsets to the first messages in log segments or the offset to the latest message on the partition.

<!-- more -->

###High Level and Simple Consumer API###

While Kafka's producer API is fairly straightforward, the consmer API is another story. There are two flavours of consumer API: **high level** and **simple**. The former, as its name suggests, provides a high level abstraction(**Consumer Group**) and just does the job(consuming messages). The latter, despite its name, is not simple at all and requires significant efforts to make it behave correctly. Lately, I have a chance to play with the simple API and feel there's something worth notice.

The general advise is to use the **high level API** whenever it's possible. A typical scenario is that consumers just want to keep reading messages sent to the topic in question. High level API on behalf of us handles the nitty-gritty of communication to brokers and leaders, plus, the offsets(the points to which we have conumed messages) are also managed for us.

Despite its convenience, there are many cases where the high level API doesn't cut. For example, re-reading messages from a particular time/offset. In addition, the fact that it would block the thread while there's nothing to read makes it difficult to use along with other concurrent constructs like Future and Actor. That's where the **simple API** to chime in.


Kafka develoers have kindly provided sample code for both kinds of API on the project wiki[1](https://cwiki.apache.org/confluence/display/KAFKA/Consumer+Group+Example)[2](https://cwiki.apache.org/confluence/display/KAFKA/0.8.0+SimpleConsumer+Example). The samples are explained in detail. With very little, if at all, modification, we can see them run with the newest release(0.8.2.1) of Kafka.

However, in the sample code for simple consumer API, the section that queries message offsets deserves a further explanation.

###OffsetRequest###
To start, we prepare OffsetRequest(the API allows us to query multiple partitions across different topics at once. For expediency's sake we stick to only one partition):

```java
// The partition of topic that we will query for
TopicAndPartition topicAndPartition =
	new TopicAndPartition(topic, partition);

Map<TopicAndPartition, PartitionOffsetRequestInfo> requestInfo =
	new HashMap<TopicAndPartition, PartitionOffsetRequestInfo>();

// whichTime??? 1???
requestInfo.put(topicAndPartition,
	new PartitionOffsetRequestInfo(whichTime, 1));

kafka.javaapi.OffsetRequest request =
	new kafka.javaapi.OffsetRequest(requestInfo,
    kafka.api.OffsetRequest.CurrentVersion(),clientName);

OffsetResponse response = consumer.getOffsetsBefore(request);
...
```

The following scala code pretty much does the same thing. I'll continue with scala code as Kafka internally is written in scala.

```scala
// The partition of topic that we will query for
val part = TopicAndPartition(topic, partitionId)

// whichTime??? 1???
val info = PartitionOffsetRequestInfo(whichTime, 1)

val infoMap = Map(part -> info)

val request = OffsetRequest(info)

val response = consumer.getOffsetsBefore(request)
...
```

Let's take a look at `PartitionOffsetRequestInfo`, it supplies a query criterion for the corresponding partition. `time` can be a unix timestamp or either one of the two special constants:

```scala
case class PartitionOffsetRequestInfo(time: Long, maxNumOffsets: Int)

val kafka.api.OffsetRequest.EarliestTime: Long = -1
val kafka.api.OffsetRequest.LatestTime: Long = -2
```

Does setting `maxNumOffset` to anything larger than 1 give us more message offsets before the given time? To answer the question we can dive into Kafka's source code. Before that, let us see how Kafka stores messages for a partition.

Generally speaking, we have a big write-ahead log to store messages for each partition. In reality, the big log is divided into several log segments and stored in different files. The default value of a segment is 1 GB but it can be configured:

```
// server.properties
# The maximum size of a log segment file. When this size is reached a new log segment will be created.
#log.segment.bytes=1073741824 # 1024*1024*1024 = 1073741824 = 1G
log.segment.bytes=65536
```

Here I change it to 64 KB for testing. **WARNING:** change in segment size would remove the existing log(hence messages are lost).


![log files on my machine](/images/post/logsegments.png)

###fetchOffsetsBefore###

Now we move on to see how offset requests are processed. Eventually, a request is dealt by `fetchOffsetBefore`:

```scala
// scala/kafka/server/KafkaApis.scala
def fetchOffsetsBefore(log: Log, timestamp: Long, maxNumOffsets: Int): Seq[Long] = {
    val segsArray = log.logSegments.toArray
    var offsetTimeArray: Array[(Long, Long)] = null
    if(segsArray.last.size > 0)
      offsetTimeArray = new Array[(Long, Long)](segsArray.length + 1)
    else
      offsetTimeArray = new Array[(Long, Long)](segsArray.length)

    for(i <- 0 until segsArray.length)
      offsetTimeArray(i) = (segsArray(i).baseOffset, segsArray(i).lastModified)
    if(segsArray.last.size > 0)
      offsetTimeArray(segsArray.length) = (log.logEndOffset, SystemTime.milliseconds)

    var startIndex = -1
    timestamp match {
      case OffsetRequest.LatestTime =>
        startIndex = offsetTimeArray.length - 1
      case OffsetRequest.EarliestTime =>
        startIndex = 0
      case _ =>
        var isFound = false
        debug("Offset time array = " + offsetTimeArray.foreach(o => "%d, %d".format(o._1, o._2)))
        startIndex = offsetTimeArray.length - 1
        while (startIndex >= 0 && !isFound) {
          if (offsetTimeArray(startIndex)._2 <= timestamp)
            isFound = true
          else
            startIndex -=1
        }
    }

    val retSize = maxNumOffsets.min(startIndex + 1)
    val ret = new Array[Long](retSize)
    for(j <- 0 until retSize) {
      ret(j) = offsetTimeArray(startIndex)._1
      startIndex -= 1
    }
    // ensure that the returned seq is in descending order of offsets
    ret.toSeq.sortBy(- _)
}
```

In this function, `segsArray: Array[LogSegment]` refers to all log segments for this partition. Whenever the log grows over the configured size, the number of log segments/files increases by 1, so does `segsArray`.

`segsArray` first gets mapped to `offsetTimeArray: Array[(Long, Long)]`:

```scala
for(i <- 0 until segsArray.length)
	offsetTimeArray(i) = (segsArray(i).baseOffset,
    					  segsArray(i).lastModified)
```

`baseOffset` is the offset of the first message(in the previous segment) in this segemnt `i`.
`lastModified` is the timestamp of the latest message in this segment `i`.

![map log segements to offsetTimeArray](/images/post/offsetrequest1.png)

In the picture, we see the first segment reaches the configured size and new messages are appended to the second segment.

- Tuple `offsetTimeArray(0)` contains the **offset to the first message** in the first segement and the **timestamp of the last** message in the the first segment.
- Tuple `offsetTimeArray(1)` contains the **offset to the first message** in the second segement and the **timestamp of the latest** message in the the second segment.


Then, a padding element is added to `offsetTimeArray`, which contains the **offset to the latest message** and **the current time**.
```scala
offsetTimeArray(segsArray.length) = (log.logEndOffset,
									 SystemTime.milliseconds)
```

![map log segements to offsetTimeArray](/images/post/offsetrequest2.png)


Next, it tries to find the `startIndex` of `offsetTimeArray`.

1. If `timestamp == LatestTime`, we gets the last index(of the padding element). 
2. If `timestamp == EarliestTime`, we get 0.
3. Otherwise, search backward to find the first element which latest message happened before `timestamp`.

The offset of the first message in the selected log segment will be returned.

```scala
    timestamp match {
      case OffsetRequest.LatestTime =>
        startIndex = offsetTimeArray.length - 1
      case OffsetRequest.EarliestTime =>
        startIndex = 0
      case _ =>
        var isFound = false
        debug("Offset time array = " + offsetTimeArray.foreach(o => "%d, %d".format(o._1, o._2)))
        startIndex = offsetTimeArray.length - 1
        while (startIndex >= 0 && !isFound) {
          if (offsetTimeArray(startIndex)._2 <= timestamp)
            isFound = true
          else
            startIndex -=1
        }
    }

```

![map log segements to offsetTimeArray](/images/post/offsetrequest3.png)

From the figure, assume `maxNumOffsets == 1`,

1. If `timestamp == LatestTime`, we get `(8, now)` and return `[8]`, which is the offset to the latest message in the partition.

2. If `timestamp == EarliestTime`, we get `(0, t1)` and return `[0]` which is the offset to the first messsage in the partition.

3. If `timestamp == T1`, we get `(0, t1)` and return `[0]`. Note that `T1 < t2`.

4. If `timestamp == T2`, we get `(7, t2)` and return `[7]`.



Finally, we are going to see what `maxNumOffsets` is for. If we're interrested more than one element(log segment), we get a chance to ask more offsets to the first messages in previous segments.

```scala
val retSize = maxNumOffsets.min(startIndex + 1)
val ret = new Array[Long](retSize)
for(j <- 0 until retSize) {
  ret(j) = offsetTimeArray(startIndex)._1
  startIndex -= 1
}
```

Say if `maxNumOffsets == 2`, then:

1. If `timestamp == LatestTime`, we get `(8, now), (7, t2)` and return `[8, 7]`.

2. If `timestamp == EarliestTime`, we only get `(0, t1)` and return `[0]`.

3. If `timestamp == T1`, we only get `(0, t1)` and return `[0]`.

4. If `timestamp == T2`, we get `(7, t2), (0, t1)` and return `[7, 0]`.

That's it. To conclude, **the offsets we get, will be either the offsets to the first messages in log segments or the offset to the latest message on the partition**.



This also explains a common problem: people often see offset 0 when they supply a unix `timestamp` to query offsets. That's simply because `log.sement.bytes` is 1 GB per default, the number of segments probably has't grown to more than 1 for the time being.







