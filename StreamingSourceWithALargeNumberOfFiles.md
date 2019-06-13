Handling streaming sources with a large number (millions) of files
============
# I. The problem
Small file is something that the Hadoop community doesn't like. It affects your solution's performance and resources ([https://medium.com/arabamlabs/small-files-in-hadoop-88708e2f6a46]())

We are in the telecoms field, with tens of thousands of network nodes. When we collect data from those nodes, it comes in tens of thousands of small files in each collection.
We don't like that, but in our case, we have to live with that. More importantly, degraded performance is not the worst that we experienced.

We built a solution which is expected to read a stream of about 50 thousand small files every 15 minutes. The first snapshot version ran well, with 4 sets of ~10K static files each. Logically, all processing was well. 40K files were there in the bucket. We turned on our processor, waited, got the result, verified the result. All good.

Now a live data test. Turned processor on, pumped the files in, got the result, checked, and, oops, about 10% of data was missing.
Now another live data test, with a more production-grade data: 10 hour worth of data - roughly 2 million files.

```puml
[hadoop@ip-10-0-0-123 ~]$ date;aws s3 ls s3://our-bucket/ | wc -l;date
Thu Oct 31 06:04:56 UTC 2018
2057717
Thu Oct 31 06:15:50 UTC 2018
```
10 minutes was the time we needed to list 2 million files on S3 from an EC2 (AWS VPC endpoint for S3 is on, so all communication is internally to AWS network)
# II. Solutions
## 1. An easy but not so nice
Pop a file from the bucket to the left, process it, then it to the bucket to the right.
So files in the left bucket are to be processed, and to the right are the processed ones. *At-least-once* is easily achieved. Some more tweaks then we could have *Exactly-once*

**What's wrong here?** <BR/>
Not much. Only one issue we could see: data is moved. What if we have another consumer of that data? Another move? Another consumer? Another Move again? <BR> That's fine. But we don't like that.
## 2. Another easy, but ...
Listing only those files which have creation time fall within a specific time window, and move the time window forward every time we list.<BR>Oh, that's good, simple, and clean. But? We are not aware of such feature in HDFS, and we were told that S3 doesn't have that either.

# III. A closer look
## 1. Dealing with missing files
### Root cause analysis
The root cause of our issue came from that "eventually consistency" behaviour of AWS S3. As per Amazon's document, this behaviour is only observable when files got updated or updated. However, they should have warned their customers that this behaviour might affect them in listing the files as well.[*]

Our experiment which proved this bad behaviour: we pumped 10000 files to S3. It took about 30 seconds to complete. In the middle of the process, we ran the 1st listing, found 5000 files, with last-modified time equal or earlier than the then timestamp, let's say: 10:10:10.123. After the client had sent all 10000 files we had another listing, trying to avoid the processed files based on that timestamp. And we found only 4900 files having timestamp later than 10:10:10.123.<BR>
Where are those 100 files?<BR>
Some of them have timestamp of 10:10:10.123, and some of them have even earlier timestamp.

The rate of missing files depends on how fast we pump the files in, and how frequently we do the listing. The faster the rate, the more missed files.

*\* Even without that eventually consistency, we still have this files missing problem. The severity would be milder though. As long as there are multiple of new files coming within a single recordable time unit (millisecond in S3 case), and we are doing the listing when that very moment has not completely passed, then we'll miss some files. <BR>
\*\* When not using S3, but some traditional file system, there's a bigger issue: file partly exists (e.g: only the first part of the file is visible)*

### Our solution
Not quite complex: create a buffer. Instead of listing files last modified from after 10:10:10.123, why not move backward some seconds, like 10:10:***08***.123?

Obstacles? There are some:
- We'll need to keep the list of files processed during those "some seconds", to maintain exactly-once capability.
- We'll need to persist that list in our check-pointing process
- We'll need to handle the case when we want to change the size of that buffer, from, let's say, 2 seconds to 5 seconds.
### Another solution
Instead of re-processing some seconds, we can have a deferred watermark. For every listing, we'll filter out the files with timestamp equal or earlier than the current watermark as well as the ones later than *(now - buffer)*. The new watermark is also set to *(now - buffer)*.

*The good*: no need to maintain the list of processed files.<BR>
*The bad*: a delay gets introduced into our pipeline.

## 2. Dealing with listing millions of files
The obvious solution is to have a hierarchical folder structure. Most intuitively, this hierarchy should be done basing on time. Something like <BR>
```
 |- today 
     |- 0000
     |- 0015
     |- 0030
     |- ...
 |- yesterday 
     |- 0000
     |- 0015
     |- 0030
     |- ...
```
### Obstacles?
- Need to define how files are put into each folder - basing on the event time (file generation time) or processing time (received time). If event time is used, then there's a need to handle late coming files.
- For the case when the pipeline is interrupted for a long time, with a heap of non-processed files queued up, there needs to be a proper handling of pagination.

# V. Implementation
The implementation here is done in Flink.

For the missing files issue, please refer to this [Pull-Request](https://github.com/apache/flink/pull/6613)

For the 2nd issue, please refer to [this class](https://github.com/lvhuyen/sdc/blob/master/src/main/scala/com/starfox/flink/source/SmallFilesReader.scala) 