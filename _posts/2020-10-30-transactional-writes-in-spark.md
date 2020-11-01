---
title:  "Transactional Writes in Spark"
excerpt: "Covering Spark's default and Databrick's Transactional Write strategies, used to write the result of a job to a destination and guarantee no partial results are left behind in case of failure."
classes: wide
categories: [spark]
toc: true
author_profile: true
---

If you write a `Dataframe` to cloud storage (Amazon S3 or Azure Storage) on Databricks platform, you will notice `_SUCCESS` , `_started_<id>` and `_commited_<id>` files within the destination directory. In this post I'll explain why these files are created, what they represent, what is a transactional write protocol and why Databricks has its own.

## Transactional Writes

Because Spark executes an application in a distributed fashion, it is impossible to atomically write the result of the job. For example, when you write a `Dataframe`, the result of the operation will be a directory with multiple files in it, one per `Dataframe`'s partition (e.g `part-00001-...`). These partition files are written by multiple Executors, as a result of their partial computation, but what if one of them fails?  

To overcome this Spark has a concept of *commit protocol*, a mechanism that knows how to write partial results and deal with success or failure of the write operation.

The *commit protocol* can be configured with `spark.sql.sources.commitProtocolClass` and by default points to the [SQLHadoopMapReduceCommitProtocol](https://github.com/apache/spark/blob/master/sql/core/src/main/scala/org/apache/spark/sql/execution/datasources/SQLHadoopMapReduceCommitProtocol.scala) implementing class.

By default, Spark creates a temporary directory and writes all the task files there. Then, it moves those files into final destination, deletes the temporary directory and creates the `_SUCCESS` file to mark the operation as successful, if `mapreduce.fileoutputcommitter.marksuccessfuljobs` is enabled. You can check more details of this process [here](https://www.waitingforcode.com/apache-spark-sql/apache-spark-success-anatomy/read). 

The problem is that this process is not actually "transactional", because if the job is aborted some of the partition files might have already been written to the final destination. This means that the `_SUCCESS` file serves basically as an indicator whether or not the consuming entities should read those files or not.

## Transactional Writes on Databricks

Spark's default *commit protocol* is fine for HDFS, because the failure window of seeing partial results is small, due to the nature of HDFS moves. However, it is not suitable for cloud native setups, e.g writing to Amazon S3.

To solve the problem of left behind partial results on job failures and other performance issues, Databricks implemented their own transactional write protocol.

You can check all the problems with Spark's default commit protocol and the details behind Databrick's custom implementation in [Transactional I:O on Cloud Storage in Databricks](https://www.youtube.com/watch?v=w1_aOPj5ILw) talk by Eric Liang and [Transactional Writes to Cloud Storage on Databricks](https://databricks.com/blog/2017/05/31/transactional-writes-cloud-storage.html) blog post.

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

[Transactional I:O on Cloud Storage in Databricks - Eric Liang](https://www.youtube.com/watch?v=w1_aOPj5ILw)

[Transactional Writes to Cloud Storage on Databricks](https://databricks.com/blog/2017/05/31/transactional-writes-cloud-storage.html)

[Notebook - Testing Transactional Writes](https://docs.databricks.com/_static/notebooks/dbio-transactional-commit.html)
