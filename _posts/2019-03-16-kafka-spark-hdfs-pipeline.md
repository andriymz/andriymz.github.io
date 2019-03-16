---
title:  "StreamSets Pipeline: Kafka Consumer, Spark Evaluator and Hadoop FS Destination"
excerpt: "In this post I'm presenting a step-by-step guide on how to develop a StreamSets pipeline with secured Kafka as source, custom Spark evaluator and HDFS as destination."
classes: wide
categories: [streamsets]
tags: [tutorial, kafka, spark, hadoop, hdfs]
---

In this post I'm presenting a step-by-step guide on how to develop a StreamSets pipeline with secured Kafka as source, custom Spark code transformer and HDFS as destination. 

Some of the problems encountered in the tested StreamSets 3.7.2 version are also addressed.

{% include figure image_path="/assets/images/streamsets/pipeline.png" %}

# Pipeline configuration

**1 -** We start by defining the basic pipeline configurations such as the pipeline title, *Cluster Yarn Streaming* execution mode and the delivery guarantee.

{% include figure image_path="/assets/images/streamsets/pipeline_config.png" %}

**2 -** In the *Cluster* tab we can also define additional *Worker Java Options* and the YARN resource pool on which StreamSets should launch our Spark Streaming application, through the `spark.yarn.queue` parameter. By default jobs run on `root.sdc` pool. 

It's irrelevant how much workers we set, since StreamSets will spawn as many workers as the number of partitions of our source Kafka topic.

{% include figure image_path="/assets/images/streamsets/pipeline_config2.png" %}

# Kafka Consumer

To configure an instance of [Kafka Consumer](https://streamsets.com/documentation/datacollector/latest/help/datacollector/UserGuide/Origins/KConsumer.html) we should:

**1 -** Choose the *Stage Library* compatible with our *Cluster Yarn Streaming* execution mode (an error pops-up if we choose a non-compatible library) and if possible which matches the Spark 2 version installed in our cluster.

{% include figure image_path="/assets/images/streamsets/kafka_config.png" %}


**2 -** Provide the basic Kafka consumer information, such as Kafka brokers, Zookeeper quorum, source topic and consumer group.

{% include figure image_path="/assets/images/streamsets/kafka_config2.png" %}

If Kafka is secured, we should grant the required *read* and *describe* privileges to the *principal* and *consumer group*:

```console
kafka-sentry --create_role --rolename streamsets
kafka-sentry --add_role_group --rolename streamsets --groupname sparker
kafka-sentry --grant_privilege_role --rolename streamsets --privilege "Host=*->Topic=streamsets_test->action=describe"
kafka-sentry --grant_privilege_role --rolename streamsets --privilege "Host=*->Topic=streamsets_test->action=read"

kafka-sentry --grant_privilege_role --rolename streamsets --privilege "Host=*->Consumergroup=streamsets_consumer_group->action=describe"
kafka-sentry --grant_privilege_role --rolename streamsets --privilege "Host=*->Consumergroup=streamsets_consumer_group->action=read"

kafka-sentry --grant_privilege_role --rolename streamsets --privilege "Host=*->Consumergroup=spark-executor-streamsets_consumer_group->action=describe"
kafka-sentry --grant_privilege_role --rolename streamsets --privilege "Host=*->Consumergroup=spark-executor-streamsets_consumer_group->action=read" 
```

Note: The last two privileges are necessary because Spark appends a prefix of *"spark-executor-"* to the provided *consumer group* ([KafkaUtils.scala at line 160](https://github.com/Stratio/spark-kafka/blob/master/src/main/scala/org/apache/spark/streaming/kafka010/KafkaUtils.scala#L160))

**3 -** On the same tab we need to define the `security.protocol` and the `sasl.kerberos.service.name` parameters, assuming that the Java TrustStore and KeyStore required for the SSL encryption have default location and password.

{% include figure image_path="/assets/images/streamsets/kafka_config3.png" %} 

**4 -** Create the *[JAAS Login Configuration](https://docs.oracle.com/javase/7/docs/technotes/guides/security/jgss/tutorials/LoginConfigFile.html)* file and place the *keytab* at the specified location:

```console
KafkaClient {
  com.sun.security.auth.module.Krb5LoginModule required
  useTicketCache=false
  useKeyTab=true
  renewTicket=true
  doNotPrompt=true
  storeKey=true
  serviceName="kafka"
  keyTab="/tmp/streamsets_test/sparker.keytab"
  principal="sparker@XPAND.COM";
};
```

**5 -** Set the *JAAS Configuration* file path, to guarantee that the created by the Data Collector JVM containers have the required security context to consume from Kafka. In Cloudera distribution environment just add the JVM flag `-Djava.security.auth.login.config=<JAAS file path>` in the Data Collector service's *Java options* parameter.

Note: Any attempt at defining the *JAAS Configuration* at the pipeline level was not successful, e.g. with the below *Worker Java Options* and *Launcher ENV* configurations. The pipeline fails on validation with the `org.apache.kafka.common.KafkaException: Jaas configuration not found` exception.

{% include figure image_path="/assets/images/streamsets/pipeline_config2.png" %}

The version of Spark installed on mine testing environment uses the 0.10.0.x Kafka version, which doesn't allow to define the *JAAS Configuration* with the provided Kafka client properties.
```console
[root@nos1 streamsets_test]# ll /opt/cloudera/parcels/SPARK2-2.2.0.cloudera2-1.cdh5.12.0.p0.232957/lib/spark2/kafka-0.10/
total 6148
-rw-r--r-- 1 root root 5156768 Jan  9  2018 kafka_2.11-0.10.0-kafka-2.1.0.jar
-rw-r--r-- 1 root root  747732 Jan  9  2018 kafka-clients-0.10.0-kafka-2.1.0.jar
-rw-r--r-- 1 root root   82123 Jan  9  2018 metrics-core-2.2.0.jar
-rw-r--r-- 1 root root  222353 Jan  9  2018 spark-streaming-kafka-0-10_2.11-2.2.0.cloudera2.jar
-rw-r--r-- 1 root root   74683 Jan  9  2018 zkclient-0.8.jar
```

In version 0.10.2.x Kafka started to support the definition of *JAAS Configuration* through the `sasl.jaas.config` parameter, however using it will only be possible in Spark versions >2.4.0.

I successfully tested consuming from Kafka using the `sasl.jaas.config` parameter in *Standalone* execution mode and CDH 6.0.0 *Stage Library*, without the need of defining the Data Collector JVM option `-Djava.security.auth.login.config=<JAAS file path>`.


# Spark Evaluator

To configure an instance of [Spark Evaluator](https://streamsets.com/documentation/datacollector/latest/help/datacollector/UserGuide/Processors/Spark.html) we should:

**1 -** Produce our custom transformation by implementing the [SparkTransformer](https://github.com/streamsets/datacollector-plugin-api/blob/master/streamsets-datacollector-spark-api/src/main/java/com/streamsets/pipeline/spark/api/SparkTransformer.java) interface available in the `streamsets-datacollector-spark-api` dependency.

```xml
<dependency>
    <groupId>com.streamsets</groupId>
    <artifactId>streamsets-datacollector-spark-api</artifactId>
    <version>3.7.2</version>
    <scope>provided</scope>
</dependency>
```
In our example we simply transform each processed string message to upper case.

```scala
class CustomTransformer extends SparkTransformer with Serializable with Loggable {
  var emptyRDD: JavaRDD[(Record, String)] = _

  override def init(session: SparkSession, parameters: util.List[String]): Unit = {
    // Create an empty JavaPairRDD to return as 'errors'
    INFO("Init called")
    emptyRDD = session.sparkContext.emptyRDD[(Record, String)]
  }

  override def transform(javaRDD: JavaRDD[Record]): TransformResult = {
    INFO(s"Transform called")

    val errors = emptyRDD
    val result = javaRDD.rdd
      .map { record =>
        INFO(s"Transforming record '$record' with type '${record.get().getType}'")
        val recordRootFieldMap = record.get().getValueAsMap
        val recordValue = recordRootFieldMap.get("text").getValueAsString
        INFO(s"Transforming value '$recordValue'")
        recordRootFieldMap.put("text", Field.create(recordValue.toUpperCase))
        record
      }

    new TransformResult(result.toJavaRDD(), new JavaPairRDD[Record, String](errors))
  }
}
```

**2 -** Choose the *Stage Library* compatible with our *Cluster Yarn Streaming* execution mode (an error pops-up if we choose a non-compatible library) and if possible which matches the Spark 2 version installed in our cluster.

{% include figure image_path="/assets/images/streamsets/config_spark_executor.png" %} 

**3 -** Add the transformer implementing JAR to the *External Libraries*.

Unfortunately, the current 3.7.2 version of StreamSets has a known bug which does not allow to upload the library on Spark Evaluator's configuration. Clicking on the upload icon simply has no effect.

{% include figure image_path="/assets/images/streamsets/spark_executor_before_installing.png" %} 

The library can still be uploaded using the *Package Manager*, by choosing the same *Stage Library* and providing the implementation JAR.

{% include figure image_path="/assets/images/streamsets/install_external_lib.png" %} 

If the error bellow pops-up it means that the *External Library* hasn't yet been set. The default library for StreamSets installed via parcel in Cloudera distribution clusters is `/opt/cloudera/parcels/STREAMSETS_DATACOLLECTOR-3.7.2/streamsets-libs-extras/`, however the `streamsets-libs-extras` directory does not exist and the `sdc` user does not have the required permissions to create it.

{% include figure image_path="/assets/images/streamsets/jar_upload_failure.png" %} 

To complete the StreamSets set-up we will need to configure the directory by following the [*"Set Up an External Directory"* documentation section](https://streamsets.com/documentation/datacollector/latest/help/datacollector/UserGuide/Configuration/ExternalLibs.html).

After uploading the JAR the Data Collector should be restarted. Another problem I noticed is that the restart is not performed by clicking on *Restart Data Collector*. The Data Collector instance stops and is not started, until we start it manually.

{% include figure image_path="/assets/images/streamsets/installed_external_lib_restart.png" %} 

If we try to remove a previously uploaded JAR then good luck, because the checkbox seems not be working as well (ugh!). Hopefully uploading a JAR with the same name replaces the existing one.

{% include figure image_path="/assets/images/streamsets/installed_external_lib.png" %} 

Additionally, it is also possible to directly replace the JAR in its directory. Removing the library also works as workaround, since it is correctly reflected in the UI.

```console
[root@nos3 sdc-extras]# tree
.
└── streamsets-datacollector-cdh_spark_2_1_r1-lib
    └── lib
        └── streamsets-spark-transformer-1.0.jar
```


**3 -** Check that the *External Library* is available in Spark Evaluator configurations.

{% include figure image_path="/assets/images/streamsets/spark_executor_after_installing.png" %} 

**4 -** Provide the package of the class implementing the *SparkTransformer*.

{% include figure image_path="/assets/images/streamsets/config_spark_executor2.png" %} 


# Hadoop FS Destination

To configure an instance of [Hadoop FS Destionation](https://streamsets.com/documentation/datacollector/latest/help/datacollector/UserGuide/Destinations/HadoopFS-destination.html) we should:

**1 -** Choose the *Stage Library* version, if possible matching the cluster's CDH version.

{% include figure image_path="/assets/images/streamsets/config_hdfs_destination.png" %} 

**2 -** Provide the Hadoop configuration directory, available in *Resources*, and check the Kerberos authentication checkbox.

{% include figure image_path="/assets/images/streamsets/config_hdfs_destination2_kerberos.png" %} 

```console
[root@nos3 lib]# tree /data/sdc/resources/hadoop-conf/
/data/sdc/resources/hadoop-conf/
├── core-site.xml -> /etc/hadoop/conf/core-site.xml
├── hdfs-site.xml -> /etc/hadoop/conf/hdfs-site.xml
├── mapred-site.xml -> /etc/hadoop/conf/mapred-site.xml
├── ssl-client.xml -> /etc/hadoop/conf/ssl-client.xml
└── yarn-site.xml -> /etc/hadoop/conf/yarn-site.xml
```


**3 -** Select the output directory. For security reasons it's suggested that the `sdc` user does not belong to *Hadoop Proxy Users* to guarantee that only the directories the logged-in user has access to can be used as output directories, not allowing any impersonation.

{% include figure image_path="/assets/images/streamsets/config_hdfs_destination3.png" %} 

**4 -** Select the data format and the *Field's* value path.

{% include figure image_path="/assets/images/streamsets/config_hdfs_destination4.png" %} 


# Running pipeline metrics

Once the pipeline is configured and passes the validation, we can start it and check its stage metrics, as shown bellow.

{% include figure image_path="/assets/images/streamsets/running.png" %} 

{% include figure image_path="/assets/images/streamsets/running2.png" %} 

{% include figure image_path="/assets/images/streamsets/running3.png" %} 


