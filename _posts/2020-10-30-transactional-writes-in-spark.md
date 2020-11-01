---
title:  "Transactional Writes in Spark"
excerpt: "Covering Spark's default and Databrick's Transactional Write strategies used to write the result of a job to a destination and guarantee no partial results are left behind in case of failure."
classes: wide
categories: [spark]
toc: true
author_profile: true
---

{% include figure image_path="/assets/images/spark/transactional_writes/intro.png" %}

If you write a `Dataframe` to cloud storage (Amazon S3 or Azure Storage) on Databricks platform, you will notice `_SUCCESS` , `_started_<id>` and `_commited_<id>` files within the destination directory. In this post I'll cover why these files are created, what they represent, what is a transactional write *commit protocol* and different implementations of it: Hadoop Commit V1, Hadoop Commit V2 and Databrick's DBIO Transactional Commit.

## Transactional Writes

Because Spark executes an application in a distributed fashion, it is impossible to atomically write the result of the job. For example, when you write a `Dataframe`, the result of the operation will be a directory with multiple files in it, one per `Dataframe`'s partition (e.g `part-00001-...`). These partition files are written by multiple Executors, as a result of their partial computation, but what if one of them fails?  

To overcome this Spark has a concept of *commit protocol*, a mechanism that knows how to write partial results and deal with success or failure of the write operation.

The *commit protocol* can be configured with `spark.sql.sources.commitProtocolClass` and by default points to the [SQLHadoopMapReduceCommitProtocol](https://github.com/apache/spark/blob/master/sql/core/src/main/scala/org/apache/spark/sql/execution/datasources/SQLHadoopMapReduceCommitProtocol.scala) implementing class, a subclass of [HadoopMapReduceCommitProtocol](https://github.com/apache/spark/blob/master/core/src/main/scala/org/apache/spark/internal/io/HadoopMapReduceCommitProtocol.scala#L140).

There are two versions of this commit algorithm, configured as `1` or `2` on `spark.hadoop.mapreduce.fileoutputcommitter.algorithm.version`.
In version 1 Spark creates a temporary directory and writes all the staging output (task) files there. Then, at the end, when all tasks compete, Spark Driver moves those files from temporary directory to the final destination, deletes the temporary directory and creates the `_SUCCESS` file to mark the operation as successful. You can check more details of this process [here](https://www.waitingforcode.com/apache-spark-sql/apache-spark-success-anatomy/read). 

The problem of version 1 is that for many output files, Driver may take a long time in the final step. Version 2 of *commit protocol* solves this slowness by writing directly the task result files to the final destination, speeding up the commit process. However, if the job is aborted it will leave behind partial task result files in the final destination. 

Spark [supports](https://issues.apache.org/jira/browse/SPARK-20107) *commit protocol* version configuration since its 2.2.0 version, and at the time of this writing Spark's latest 3.0.1 version has a default value for it depending on running environment (Hadoop version).

{% include figure image_path="/assets/images/spark/transactional_writes/spark_protocol_version_conf.png" %}

## Transactional Writes on Databricks

As we previously saw, Spark's default *commit protocol* version 1 should be used for safety (no partial results) and version 2 for performance. However, if we opt for data safety version 1 is not suitable for cloud native setups, e.g writing to Amazon S3, due to differences cloud object stores have from real filesystems, as explained in [Spark's cloud integration guide](https://github.com/apache/spark/blob/32a0451376ab775fdd4ac364388e46179d9ee550/docs/cloud-integration.md). 

To solve the problem of left behind partial results on job failures and performance issues, Databricks implemented their own transactional write protocol called *DBIO Transactional Commit*.

You can check all the problems with Spark's default *commit protocol* and the details behind Databrick's custom implementation in [Transactional I:O on Cloud Storage in Databricks](https://www.youtube.com/watch?v=w1_aOPj5ILw) talk by Eric Liang and [Transactional Writes to Cloud Storage on Databricks](https://databricks.com/blog/2017/05/31/transactional-writes-cloud-storage.html) blog post.

Let's take a look with examples how transactional writes work on Databricks, implemented with the *commit procotol* below:

{% include figure image_path="/assets/images/spark/transactional_writes/0.png" %}

By writing the `Dataframe` to a directory, mount point of an Azure Storage container, the following files were created:

{% include figure image_path="/assets/images/spark/transactional_writes/1.png" %}

For each transaction, a unique transaction id `<tid>` is generated. At the start of each transaction, Spark creates an empty `_started_<tid>` file. If a transaction is successful, a `_committed_<tid>` file is created. Notice that each partition file also has the transaction identifier in it.

The content of the committed file for this transaction contains all the added partition files, as follows: 

```
%fs head --maxBytes=10000 /mnt/users/anzabol/data/_committed_7628641425835151768
```

```json
{
  "added": [
    "part-00000-tid-7628641425835151768-a8c83893-f6af-49ab-9e0d-8e1181b7e684-37-1-c000.snappy.parquet",
    "part-00001-tid-7628641425835151768-a8c83893-f6af-49ab-9e0d-8e1181b7e684-38-1-c000.snappy.parquet",
    "part-00002-tid-7628641425835151768-a8c83893-f6af-49ab-9e0d-8e1181b7e684-39-1-c000.snappy.parquet"
  ],
  "removed": []
}
```

After writing another `Dataframe` to the same destination, with the *append* mode, the respective transaction files were created together with the partition files. 

{% include figure image_path="/assets/images/spark/transactional_writes/2.png" %}

Once again, the committed file should contain all the files that were written during that transaction: 

```json
{
  "added": [
    "part-00000-tid-3452754969657447620-98b3663b-fbe5-49c1-bbbc-9d0a2413fc20-44-1-c000.snappy.parquet",
    "part-00001-tid-3452754969657447620-98b3663b-fbe5-49c1-bbbc-9d0a2413fc20-45-1-c000.snappy.parquet"
  ],
  "removed": []
}
```

My last test consisted in writing again the first `Dataframe` but with *overwrite* mode, to see what happens. In this case the committed file looked like this:

```json
{
  "added": [
    "part-00000-tid-5097121921988240910-6ec8b872-eaca-4dd8-83b6-0822a8a18189-50-1-c000.snappy.parquet",
    "part-00001-tid-5097121921988240910-6ec8b872-eaca-4dd8-83b6-0822a8a18189-51-1-c000.snappy.parquet",
    "part-00002-tid-5097121921988240910-6ec8b872-eaca-4dd8-83b6-0822a8a18189-52-1-c000.snappy.parquet"
  ],
  "removed": [
    "_SUCCESS",
    "part-00000-tid-3452754969657447620-98b3663b-fbe5-49c1-bbbc-9d0a2413fc20-44-1-c000.snappy.parquet",
    "part-00000-tid-7628641425835151768-a8c83893-f6af-49ab-9e0d-8e1181b7e684-37-1-c000.snappy.parquet",
    "part-00001-tid-3452754969657447620-98b3663b-fbe5-49c1-bbbc-9d0a2413fc20-45-1-c000.snappy.parquet",
    "part-00001-tid-7628641425835151768-a8c83893-f6af-49ab-9e0d-8e1181b7e684-38-1-c000.snappy.parquet",
    "part-00002-tid-7628641425835151768-a8c83893-f6af-49ab-9e0d-8e1181b7e684-39-1-c000.snappy.parquet"
  ]
} 
```

As you can see, all the existing partition and success files were deleted, and the partition files of the written `Dataframe` added.

Since each successful transaction leaves a committed file, when reading data Spark ignores all the files that contain a `<tid>` for which there is no commit file. This way there is no way of reading corrupted / dirty data.

To clean up partition files of uncommitted transactions, there is [VACUUM](https://docs.databricks.com/spark/latest/spark-sql/dbio-commit.html?_ga=2.110458629.1898107027.1604185218-1080905499.1603716625#clean-up-uncommitted-files) operation, that accepts an output path and the retention period. 


## Resources

[Apache Spark's _SUCESS anatomy](https://www.waitingforcode.com/apache-spark-sql/apache-spark-success-anatomy/read)

[What is the difference between mapreduce.fileoutputcommitter.algorithm.version=1 and 2](http://www.openkb.info/2019/04/what-is-difference-between.html)

[Transactional I:O on Cloud Storage in Databricks - Eric Liang](https://www.youtube.com/watch?v=w1_aOPj5ILw)

[Transactional Writes to Cloud Storage on Databricks](https://databricks.com/blog/2017/05/31/transactional-writes-cloud-storage.html)

[Notebook - Testing Transactional Writes](https://docs.databricks.com/_static/notebooks/dbio-transactional-commit.html)

[Spark 2.0.0 cluster takes a long time to append data](https://kb.databricks.com/data/append-slow-with-spark-2.0.0.html)
