Streaming frameworks
====================

In this page, we will be considering the supports for processing data streams by
Spark and Flink. Conclusion is at the end. This comparison is done for my own
personal purpose, basing purely on my knowledge and experience using these
Apache products. I must not be blamed for any bad consequence, damages happen at
your side during and after reading this page I am associated with neither Spark
nor Flink team.

On the other hand, this comparison is on the ground that I am already familiar
with Spark, with most of my (batch) Spark jobs written in Python.

# 1. General view

Spark
-----

With our current situation, Spark Streaming has one big "pros": language
friendly. Spark team has tried to use the same terminology between streaming and
batch processing. So the methods to read data from sources (until now we have
used files and Hive/Glue tables), to transform data, and to write data to sinks,
would have similar (if not same) signatures as with batch processing. 

### Streaming vs. Structured Streaming

On Spark side, there are two different options: Spark Streaming
[[1]](https://spark.apache.org/docs/latest/streaming-programming-guide.html) (which
works with streams of data called DStream, or Discretized Stream) and Spark
Structured Streaming
[[2]](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html).

First: DStream vs. StructuredStream is like RDD vs DataFrame/DataSet. With
Structured Streaming, we can use SQL, and we have optimizer runs in the
background. 

Second: there is nothing called "event time" in DStream. This also means there's
no late-data - all records are on-time.  

There seems to be not much difference between DStream and Kafka Streaming, or
Kinesis/Firehose, or using AWS lambda to process discrete events.

### Python vs. Scala

With Spark Structured Streaming, either Python or Scala can be used. With the
current Spark version
(2.4.3)[[2]](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html),
except for stateful operations (which will be discussed in more detail
below), there is no gap between out-of-the-box features supported for Python and
Scala.

However, when we need something different (e.g. a custom file source, compressed
using zip, and when uncompressed, has custom header and footer), as there's no
out-of-the-box support for reading binary files, then we would need to customize
the source function. With Python, it seems we are out of luck here.

Flink
-----

### Low-level vs. high-level API

Flink has some different sets of high-level API, notably Table and CEP (Complex
Events Processing).

With Table API, from the usage point of view, we have something similar to Spark
Structured Streaming. Data streams are viewed as dynamic/unbounded tables.
Transformations/SQLs are also similar to Spark's counterparts. A bit more hassle
(some more lines of code) when converting streams from/to tables.

CEP makes it easy to define patterns in incoming events in streams. The
"Concierge" problem would be a good candidate for using CEP: rules like "sum of
counter.x in the last 72 hours \< X and every counter.y in the last 2 hours \>
Y" can be represented in CEP with half the number of lines in SQL. The
not-quite-useful aspect of CEP, as of now, is that it is not (yet) possible to
externalize CEP rules as we can do with SQL.

Low-level API is what Flink excels.

### Python vs Scala

Flink is written mostly in Java, and provides the Java API set and Scala API
set. As the nature of Scala language, the Scala API set is the *superset* of the
Java one in term of capability.

#### Python API

Python is still something considered "experimental" in Flink. It is a low-level
API set, built on top, and is a *subset* of the Java one.

There* *is not enough document regarding how to expand/enhance this API set,
thus it is not possible to say that this Flink Python API provides anything
better than Python Spark Structured Streaming.

#### Beam

Beam is a cross-language programming model to define and execute data processing
pipelines (batch and streaming), which acts as a wrapper for
Flink/Spark/DataFlow/... The target is to be able to run the same pipeline on
either on Spark or Flink or ... However, as of now, the support for Spark is
minimal, while Apache Flink and Google DataFlow are dominant in the
feature-matrix
board [[3](https://beam.apache.org/documentation/runners/capability-matrix/)].

A Beam pipeline can be written in Java (or any JVM based languages, like Scala),
Python, or Go. Current features set is different, richest in Java, and second in
Python. Cross-language is supported, which means that we can use both Java and
Python in the same pipeline.

One more problem with Beam, as of now, is the amount of work needed to automate
the deployment. As it is not available in AWS EMR yet, we will need to do all
the task of installing, running the JobService, and triggering the pipeline
manually.

# 2. Data Transformation

When looking at a streaming problem, people tend to look for answers for these 4
questions:

-   *'What'* results are calculated?

-   *'Where'* in event time are results calculated?

-   *'When'* in processing time are results materialized?

-   *'How'* do the refinements of results relate?

Thus, when looking for a streaming framework, we will look at how each framework
answers those 4 questions.

*Where* in event time are results calculated?
---------------------------------------------

This is the functionality which the older Spark Streaming completely lacks.
Spark Structured Streaming came to solve it, and it is doing rather well here.
The only main feature that it lacks is *Session Window*, which, for ETL jobs, we
are not going to need one in the foreseeable future. 

*What* results are calculated?
------------------------------

In term of data transformation and aggregation, the biggest advantage of Flink
over Spark is states management.

Spark Structured Streaming does support stateful operations. However, it is
limited to the two methods: MapGroupWithState and FlatmapGroupWithState, which
is convenient for doing natural groupBy operations, but would need some dev
effort when working with non-natural ones (e.g the requirement to enrich one
fast stream with one slow stream).

With PySpark, the problem becomes worse, as there is no mention of those two
methods anywhere in Spark documents. To solve the above mentioned enrichment
issue, the only solution is to convert the slow stream to a static dataset. This
means that we need to pause the pipeline every now and then, reload that slow
dataset, and resume the processing.

Moreover, Spark currently poses many cross-dependencies between transforming
data, especially when stateful operations are used, and emitting outputs (the
when & how questions) which are mentioned explicitly in Spark's official
document. This would reduce dramatically Spark's potential in solving our
problem (ref. "Join Operation" in
[[2]](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html) and <https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#unsupported-operations>).

*When* in processing time are results materialized?
---------------------------------------------------

This is where Spark is weak at. It gives only two fixed options: either
(1) when crossing the watermark, or (2) on every new message.

One example of how this would affect our business logic is calculating the
rolling average of a counter in the last 8 data points:

-   If we choose option (2), then for each sliding window of 2 hours, there will
    be 8 output records (one for the first input record, one for the first 2
    input records,...).

-   If we choose option (1), as there's no natural marker to tell the stream
    processor that "all data for that timestamp 10:45 has come", the problem is
    more severe:

    -   If we just discard the late coming data (which currently accounts for
        about 10% of data), all aggregated output for 10:45 will come at 11:00.
        This means 15-minute delay for the happy scenario, and
        less-than-8-data-point for the data-arriving-late scenario.

    -   If we wait for the late date, let's say, for 1 hour, then the first (and
        all) output for 10:45 will come at 12:00.

Moreover, as mentioned earlier, not all operations give the flexibility of
choosing between (1) or (2).

Flink, on the other hand, supports all the flexibility on when to trigger the
output. However, this is with the Java/Scala API. No document for the Python API
has been found for this.

*How* do the refinements of results relate?
-------------------------------------------

The support for this is similar between Spark and Flink. Spark lack the
"Retract" mode. However, with the support of state management, another layer of
aggregation would solve the problem 

# 3. Connectors

Sources
-------

Spark currently supports Kafka and Files sources. With file sources, as
mentioned above, only a few pre-defined formats are supported. Adding new
sources is possible with Scala, but with Python is a myth. There's no document
mentioning how to implement this functionality with PySpark.

Adding an async operation is currently not supported in Spark (an example
use-case is to query an external database, or to query Splunk for some
subscribers,...)

Flink supports some more type of sources, notably Kinesis.

Sinks
-----

With the support of *ForEach *function for Python starting from Spark 2.4, it is
possible to say that there's no limitation with Spark in this aspect.

# 4. Performance

There have been some benchmarks regarding the performance between the
proprietary optimized version of Spark (by DataBricks) and Flink, but none was
done by a non-bias 3rd party.

In term of resource consumption (memory/storage), with the support of
incremental aggregation, Flink clearly has an advantage. For example for the
Concierge problem, where the rolling aggregation of 72-hour data is needed, with
Flink, only 1 cached aggregated object is needed for each window. While with
non-incremental aggregation, on average, 36 hours of data is needed to be stored
in memory.

# 5. Auto-scaling

Spark is ahead of Flink in this aspect, with the support for auto-scaling. 

In Flink, to scale up/down, there's a need to stop the job with a savepoint,
change the configuration, and resume from that savepoint.

*Edited*: *AWS provides a managed environment for running Flink jobs, which is
expected to take all the hassle of streaming jobs management away, which would
bring Flink forward of Spark Streaming in this aspect. *

# 6. Conclusion

Python or Scala/Java?
---------------------

Python seems to be a no-go for now, because of lacking:

-   Stateful aggregation in Spark

-   Custom triggers (the when question).

-   Custom source, either in Spark or Flink

Even with the use of Beam and Flink, there is still a need for mixing Python
with Java/Scala.

Spark or Flink?
--------------

In our situation, the biggest cons of Spark are:

-   Firstly, the flexibility in generating outputs from window-based operations
    (again, that when question) - think of the solution for rolling average.

-   A bit less severe is stateful operations - think of the solution for data
    enrichment with THOR FLS

-   And, the third point, is Kinesis source.

Pros of Spark: the learning curve - due to the similarity between batch and
streaming, terminology and methods signature.

Reference
=========

[1] <https://spark.apache.org/docs/latest/streaming-programming-guide.html>

[2] <https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html>

[3] <https://beam.apache.org/documentation/runners/capability-matrix/>
