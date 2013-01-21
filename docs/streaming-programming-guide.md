---
layout: global
title: Spark Streaming Programming Guide
---

* This will become a table of contents (this text will be scraped).
{:toc}

# Overview
A Spark Streaming application is very similar to a Spark application; it consists of a *driver program* that runs the user's `main` function and continuous executes various *parallel operations* on input streams of data. The main abstraction Spark Streaming provides is a *discretized stream* (DStream), which is a continuous sequence of RDDs (distributed collection of elements) representing a continuous stream of data. DStreams can created from live incoming data (such as data from a socket, Kafka, etc.) or it can be generated by transformation of existing DStreams using parallel operators like map, reduce, and window. The basic processing model is as follows: 
(i) While a Spark Streaming driver program is running, the system receives data from various sources and and divides the data into batches. Each batch of data is treated as a RDD, that is a immutable and parallel collection of data. These input data RDDs are automatically persisted in memory (serialized by default) and replicated to two nodes for fault-tolerance. This sequence of RDDs is collectively referred to as an InputDStream.
(ii) Data received by InputDStreams are processed processed using DStream operations. Since all data is represented as RDDs and all DStream operations as RDD operations, data is automatically recovered in the event of node failures.  

This guide shows some how to start programming with DStreams. 

# Initializing Spark Streaming
The first thing a Spark Streaming program must do is create a `StreamingContext` object, which tells Spark how to access a cluster. A `StreamingContext` can be created by using

{% highlight scala %}
new StreamingContext(master, jobName, batchDuration)
{% endhighlight %}

The `master` parameter is either the [Mesos master URL](running-on-mesos.html) (for running on a cluster)or the special "local" string (for local mode) that is used to create a Spark Context. For more information about this please refer to the [Spark programming guide](scala-programming-guide.html). The `jobName` is the name of the streaming job, which is the same as the jobName used in SparkContext. It is used to identify this job in the Mesos web UI. The `batchDuration` is the size of the batches (as explained earlier). This must be set carefully such the cluster can keep up with the processing of the data streams. Starting with something conservative like 5 seconds maybe a good start. See [Performance Tuning](#setting-the-right-batch-size) section for a detailed discussion.

This constructor creates a SparkContext object using the given `master` and `jobName` parameters. However, if you already have a SparkContext or you need to create a custom SparkContext by specifying list of JARs, then a StreamingContext can be created from the existing SparkContext, by using
{% highlight scala %}
new StreamingContext(sparkContext, batchDuration)
{% endhighlight %}



# Attaching Input Sources - InputDStreams
The StreamingContext is used to creating InputDStreams from input sources:

{% highlight scala %}
// Assuming ssc is the StreamingContext
ssc.networkStream(hostname, port)    // Creates a stream that uses a TCP socket to read data from hostname:port
ssc.textFileStream(directory)   // Creates a stream by monitoring and processing new files in a HDFS directory
{% endhighlight %}

A complete list of input sources is available in the [StreamingContext API documentation](api/streaming/index.html#spark.streaming.StreamingContext). Data received from these sources can be processed using DStream operations, which are explained next.



# DStream Operations
Once an input DStream has been created, you can transform it using _DStream operators_. Most of these operators return new DStreams which you can further transform. Eventually, you'll need to call an _output operator_, which forces evaluation of the DStream by writing data out to an external source.

## Transformations

DStreams support many of the transformations available on normal Spark RDD's:

<table class="table">
<tr><th style="width:25%">Transformation</th><th>Meaning</th></tr>
<tr>
  <td> <b>map</b>(<i>func</i>) </td>
  <td> Returns a new DStream formed by passing each element of the source through a function <i>func</i>. </td>
</tr>
<tr>
  <td> <b>filter</b>(<i>func</i>) </td>
  <td> Returns a new stream formed by selecting those elements of the source on which <i>func</i> returns true. </td>
</tr>
<tr>
  <td> <b>flatMap</b>(<i>func</i>) </td>
  <td> Similar to map, but each input item can be mapped to 0 or more output items (so <i>func</i> should return a Seq rather than a single item). </td>
</tr>
<tr>
  <td> <b>mapPartitions</b>(<i>func</i>) </td>
  <td> Similar to map, but runs separately on each partition (block) of the DStream, so <i>func</i> must be of type
    Iterator[T] => Iterator[U] when running on an DStream of type T. </td>
</tr>
<tr>
  <td> <b>union</b>(<i>otherStream</i>) </td>
  <td> Return a new stream that contains the union of the elements in the source stream and the argument. </td>
</tr>
<tr>
  <td> <b>groupByKey</b>([<i>numTasks</i>]) </td>
  <td> When called on a stream of (K, V) pairs, returns a stream of (K, Seq[V]) pairs. <br />
<b>Note:</b> By default, this uses only 8 parallel tasks to do the grouping. You can pass an optional <code>numTasks</code> argument to set a different number of tasks.
</td>
</tr>
<tr>
  <td> <b>reduceByKey</b>(<i>func</i>, [<i>numTasks</i>]) </td>
  <td> When called on a stream of (K, V) pairs, returns a stream of (K, V) pairs where the values for each key are aggregated using the given reduce function. Like in <code>groupByKey</code>, the number of reduce tasks is configurable through an optional second argument. </td>
</tr>
<tr>
  <td> <b>join</b>(<i>otherStream</i>, [<i>numTasks</i>]) </td>
  <td> When called on streams of type (K, V) and (K, W), returns a stream of (K, (V, W)) pairs with all pairs of elements for each key. </td>
</tr>
<tr>
  <td> <b>cogroup</b>(<i>otherStream</i>, [<i>numTasks</i>]) </td>
  <td> When called on DStream of type (K, V) and (K, W), returns a DStream of (K, Seq[V], Seq[W]) tuples.</td>
</tr>
<tr>
  <td> <b>reduce</b>(<i>func</i>) </td>
  <td> Returns a new DStream of single-element RDDs by aggregating the elements of the stream using a function func (which takes two arguments and returns one). The function should be associative so that it can be computed correctly in parallel. </td>
</tr>
<tr>
  <td> <b>transform</b>(<i>func</i>) </td>
  <td> Returns a new DStream by applying func (a RDD-to-RDD function) to every RDD of the stream. This can be used to do arbitrary RDD operations on the DStream. </td>
</tr>
</table>

Spark Streaming features windowed computations, which allow you to report statistics over a sliding window of data. All window functions take a <i>windowDuration</i>, which represents the width of the window and a <i>slideTime</i>, which represents the frequency during which the window is calculated.

<table class="table">
<tr><th style="width:25%">Transformation</th><th>Meaning</th></tr>
<tr>
  <td> <b>window</b>(<i>windowDuration</i>, </i>slideTime</i>) </td>
  <td> Return a new stream which is computed based on windowed batches of the source stream. <i>windowDuration</i> is the width of the window and <i>slideTime</i> is the frequency during which the window is calculated. Both times must be multiples of the batch interval.
  </td>
</tr>
<tr>
  <td> <b>countByWindow</b>(<i>windowDuration</i>, </i>slideTime</i>) </td>
  <td> Return a sliding count of elements in the stream. <i>windowDuration</i> and <i>slideDuration</i> are exactly as defined in <code>window()</code>.
  </td>
</tr>
<tr>
  <td> <b>reduceByWindow</b>(<i>func</i>, <i>windowDuration</i>, </i>slideDuration</i>) </td>
  <td> Return a new single-element stream, created by aggregating elements in the stream over a sliding interval using <i>func</i>. The function should be associative so that it can be computed correctly in parallel. <i>windowDuration</i> and <i>slideDuration</i> are exactly as defined in <code>window()</code>.
  </td>
</tr>
<tr>
  <td> <b>groupByKeyAndWindow</b>(windowDuration, slideDuration, [<i>numTasks</i>])
  </td>
  <td> When called on a stream of (K, V) pairs, returns a stream of (K, Seq[V]) pairs over a sliding window. <br />
<b>Note:</b> By default, this uses only 8 parallel tasks to do the grouping. You can pass an optional <code>numTasks</code> argument to set a different number of tasks. <i>windowDuration</i> and <i>slideDuration</i> are exactly as defined in <code>window()</code>.
</td>
</tr>
<tr>
  <td> <b>reduceByKeyAndWindow</b>(<i>func</i>, [<i>numTasks</i>]) </td>
  <td> When called on a stream of (K, V) pairs, returns a stream of (K, V) pairs where the values for each key are aggregated using the given reduce function over batches within a sliding window. Like in <code>groupByKeyAndWindow</code>, the number of reduce tasks is configurable through an optional second argument. 
 <i>windowDuration</i> and <i>slideDuration</i> are exactly as defined in <code>window()</code>.
</td> 
</tr>
<tr>
  <td> <b>countByKeyAndWindow</b>([<i>numTasks</i>]) </td>
  <td> When called on a stream of (K, V) pairs, returns a stream of (K, Int) pairs where the values for each key are the count within a sliding window. Like in <code>countByKeyAndWindow</code>, the number of reduce tasks is configurable through an optional second argument. 
 <i>windowDuration</i> and <i>slideDuration</i> are exactly as defined in <code>window()</code>.
</td> 
</tr>

</table>

A complete list of DStream operations is available in the API documentation of [DStream](api/streaming/index.html#spark.streaming.DStream) and [PairDStreamFunctions](api/streaming/index.html#spark.streaming.PairDStreamFunctions).

## Output Operations
When an output operator is called, it triggers the computation of a stream. Currently the following output operators are defined:

<table class="table">
<tr><th style="width:25%">Operator</th><th>Meaning</th></tr>
<tr>
  <td> <b>foreach</b>(<i>func</i>) </td>
  <td> The fundamental output operator. Applies a function, <i>func</i>, to each RDD generated from the stream. This function should have side effects, such as printing output, saving the RDD to external files, or writing it over the network to an external system. </td>
</tr>

<tr>
  <td> <b>print</b>() </td>
  <td> Prints first ten elements of every batch of data in a DStream on the driver. </td>
</tr>

<tr>
  <td> <b>saveAsObjectFiles</b>(<i>prefix</i>, [<i>suffix</i>]) </td>
  <td> Save this DStream's contents as a <code>SequenceFile</code> of serialized objects. The file name at each batch interval is generated based on <i>prefix</i> and <i>suffix</i>: <i>"prefix-TIME_IN_MS[.suffix]"</i>.
  </td>
</tr>

<tr>
  <td> <b>saveAsTextFiles</b>(<i>prefix</i>, [<i>suffix</i>]) </td>
  <td> Save this DStream's contents as a text files. The file name at each batch interval is generated based on <i>prefix</i> and <i>suffix</i>: <i>"prefix-TIME_IN_MS[.suffix]"</i>. </td>
</tr>

<tr>
  <td> <b>saveAsHadoopFiles</b>(<i>prefix</i>, [<i>suffix</i>]) </td>
  <td> Save this DStream's contents as a Hadoop file. The file name at each batch interval is generated based on <i>prefix</i> and <i>suffix</i>: <i>"prefix-TIME_IN_MS[.suffix]"</i>. </td>
</tr>

</table>

## DStream Persistence
Similar to RDDs, DStreams also allow developers to persist the stream's data in memory. That is, using `persist()` method on a DStream would automatically persist every RDD of that DStream in memory. This is useful if the data in the DStream will be computed multiple times (e.g., multiple DStream operations on the same data). For window-based operations like `reduceByWindow` and `reduceByKeyAndWindow` and state-based operations like `updateStateByKey`, this is implicitly true. Hence, DStreams generated by window-based operations are automatically persisted in memory, without the developer calling `persist()`.

Note that, unlike RDDs, the default persistence level of DStreams keeps the data serialized in memory. This is further discussed in the [Performance Tuning](#memory-tuning) section. More information on different persistence levels can be found in [Spark Programming Guide](scala-programming-guide.html#rdd-persistence).

# Starting the Streaming computation
All the above DStream operations are completely lazy, that is, the operations will start executing only after the context is started by using
{% highlight scala %}
ssc.start()
{% endhighlight %}

Conversely, the computation can be stopped by using
{% highlight scala %}
ssc.stop()
{% endhighlight %}

# Example - NetworkWordCount.scala
A good example to start off is the spark.streaming.examples.NetworkWordCount. This example counts the words received from a network server every second. Given below is the relevant sections of the source code. You can find the full source code in <Spark repo>/streaming/src/main/scala/spark/streaming/examples/WordCountNetwork.scala.

{% highlight scala %}
import spark.streaming.{Seconds, StreamingContext}
import spark.streaming.StreamingContext._
...

// Create the context and set up a network input stream to receive from a host:port
val ssc = new StreamingContext(args(0), "NetworkWordCount", Seconds(1))
val lines = ssc.networkTextStream(args(1), args(2).toInt)

// Split the lines into words, count them, and print some of the counts on the master
val words = lines.flatMap(_.split(" "))
val wordCounts = words.map(x => (x, 1)).reduceByKey(_ + _)
wordCounts.print()

// Start the computation
ssc.start()
{% endhighlight %}

To run this example on your local machine, you need to first run a Netcat server by using

{% highlight bash %}
$ nc -lk 9999
{% endhighlight %}

Then, in a different terminal, you can start NetworkWordCount by using

{% highlight bash %}
$ ./run spark.streaming.examples.NetworkWordCount local[2] localhost 9999
{% endhighlight %}

This will make NetworkWordCount connect to the netcat server. Any lines typed in the terminal running the netcat server will be counted and printed on screen.

<table>
<td>
{% highlight bash %}
# TERMINAL 1
# RUNNING NETCAT

$ nc -lk 9999
hello world





...
{% endhighlight %}
</td>
<td>
{% highlight bash %}
# TERMINAL 2: RUNNING NetworkWordCount
...
2012-12-31 18:47:10,446 INFO SparkContext: Job finished: run at ThreadPoolExecutor.java:886, took 0.038817 s
-------------------------------------------
Time: 1357008430000 ms
-------------------------------------------
(hello,1)
(world,1)

2012-12-31 18:47:10,447 INFO JobManager: Total delay: 0.44700 s for job 8 (execution: 0.44000 s)
...
{% endhighlight %}
</td>
</table>



# Performance Tuning
Getting the best performance of a Spark Streaming application on a cluster requires a bit of tuning. This section explains a number of the parameters and configurations that can tuned to improve the performance of you application. At a high level, you need to consider two things:
<ol>
<li>Reducing the processing time of each batch of data by efficiently using cluster resources.</li>
<li>Setting the right batch size such that the data processing can keep up with the data ingestion.</li>
</ol>

## Reducing the Processing Time of each Batch
There are a number of optimizations that can be done in Spark to minimize the processing time of each batch. These have been discussed in detail in [Tuning Guide](tuning.html). This section highlights some of the most important ones.

### Level of Parallelism
Cluster resources maybe underutilized if the number of parallel tasks used in any stage of the computation is not high enough. For example, for distributed reduce operations like `reduceByKey` and `reduceByKeyAndWindow`, the default number of parallel tasks is 8. You can pass the level of parallelism as an argument (see the [`spark.PairDStreamFunctions`](api/streaming/index.html#spark.PairDStreamFunctions) documentation), or set the system property `spark.default.parallelism` to change the default.

### Data Serialization
The overhead of data serialization can be significant, especially when sub-second batch sizes are to be achieved. There are two aspects to it.
* Serialization of RDD data in Spark: Please refer to the detailed discussion on data serialization in the [Tuning Guide](tuning.html). However, note that unlike Spark, by default RDDs are persisted as serialized byte arrays to minimize pauses related to GC.
* Serialization of input data: To ingest external data into Spark, data received as bytes (say, from the network) needs to deserialized from bytes and re-serialized into Spark's serialization format. Hence, the deserialization overhead of input data may be a bottleneck.

### Task Launching Overheads
If the number of tasks launched per second is high (say, 50 or more per second), then the overhead of sending out tasks to the slaves maybe significant and will make it hard to achieve sub-second latencies. The overhead can be reduced by the following changes:
* Task Serialization: Using Kryo serialization for serializing tasks can reduced the task sizes, and therefore reduce the time taken to send them to the slaves.
* Execution mode: Running Spark in Standalone mode or coarse-grained Mesos mode leads to better task launch times than the fine-grained Mesos mode. Please refer to the [Running on Mesos guide](running-on-mesos.html) for more details.
These changes may reduce batch processing time by 100s of milliseconds, thus allowing sub-second batch size to be viable.

## Setting the Right Batch Size
For a Spark Streaming application running on a cluster to be stable, the processing of the data streams must keep up with the rate of ingestion of the data streams. Depending on the type of computation, the batch size used may have significant impact on the rate of ingestion that can be sustained by the Spark Streaming application on a fixed cluster resources. For example, let us consider the earlier WordCountNetwork example. For a particular data rate, the system may be able to keep up with reporting word counts every 2 seconds (i.e., batch size of 2 seconds), but not every 500 milliseconds.

A good approach to figure out the right batch size for your application is to test it with a conservative batch size (say, 5-10 seconds) and a low data rate. To verify whether the system is able to keep up with data rate, you can check the value of the end-to-end delay experienced by each processed batch (in the Spark master logs, find the line having the phrase "Total delay"). If the delay is maintained to be less than the batch size, then system is stable. Otherwise, if the delay is continuously increasing, it means that the system is unable to keep up and it therefore unstable. Once you have an idea of a stable configuration, you can try increasing the data rate and/or reducing the batch size. Note that momentary increase in the delay due to temporary data rate increases maybe fine as long as the delay reduces back to a low value (i.e., less than batch size).

## 24/7 Operation
By default, Spark does not forget any of the metadata (RDDs generated, stages processed, etc.). But for a Spark Streaming application to operate 24/7, it is necessary for Spark to do periodic cleanup of it metadata. This can be enabled by setting the Java system property `spark.cleaner.delay` to the number of minutes you want any metadata to persist. For example, setting `spark.cleaner.delay` to 10 would cause Spark periodically cleanup all metadata and persisted RDDs that are older than 10 minutes. Note, that this property needs to be set before the SparkContext is created.

This value is closely tied with any window operation that is being used. Any window operation would require the input data to be persisted in memory for at least the duration of the window. Hence it is necessary to set the delay to at least the value of the largest window operation used in the Spark Streaming application. If this delay is set too low, the application will throw an exception saying so.

## Memory Tuning
Tuning the memory usage and GC behavior of Spark applications have been discussed in great detail in the [Tuning Guide](tuning.html). It is recommended that you read that. In this section, we highlight a few customizations that are strongly recommended to minimize GC related pauses in Spark Streaming applications and achieving more consistent batch processing times.

* <b>Default persistence level of DStreams</b>: Unlike RDDs, the default persistence level of DStreams serializes the data in memory (that is, [StorageLevel.MEMORY_ONLY_SER](api/core/index.html#spark.storage.StorageLevel$) for DStream compared to [StorageLevel.MEMORY_ONLY](api/core/index.html#spark.storage.StorageLevel$) for RDDs). Even though keeping the data serialized incurs a higher serialization overheads, it significantly reduces GC pauses.

* <b>Concurrent garbage collector</b>: Using the concurrent mark-and-sweep GC further minimizes the variability of GC pauses. Even though concurrent GC is known to reduce the overall processing throughput of the system, its use is still recommended to achieve more consistent batch processing times.

# Master Fault-tolerance (Alpha)
TODO

* Checkpointing of DStream graph

* Recovery from master faults

* Current state and future directions