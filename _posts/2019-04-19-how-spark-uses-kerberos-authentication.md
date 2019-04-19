---
title:  "How Spark Uses Kerberos Authentication"
excerpt: "Have you ever wondered how Spark uses Kerberos authentication? How and when the provided through the spark-submit *--principal* and *--keytab* options are used? This post answers those questions and explains all the magic that happens 'under the hood' when you submit your long-running applications."
classes: wide
categories: [kerberos]
toc: true
tags: [spark, hadoop security, hadoop, security]
---

Have you ever wondered how Spark uses Kerberos authentication? How and when the provided through the spark-submit `--principal` and `--keytab` options are used? This post answers those questions and explains all the magic that happens "under the hood" when you submit your long-running applications.

The post's content is based on my source code analysis of Spark version 2.1.0, so other versions of Spark may have some slight changes, but I assume the overall behaviour is the same because the delegation token renewal logic was only migrated from Application Master to the Driver in Spark 3.0 (see [Jira issue](https://issues.apache.org/jira/browse/SPARK-25689)), which hasn't been released yet.

# Application Submission by the Client

My [last post](/kerberos/authentication-using-kerberos/) about Kerberos briefly explains what delegation tokens are and how the [Credential](https://hadoop.apache.org/docs/current/api/org/apache/hadoop/security/Credentials.html) objects containing them can be stored to and read from HDFS, allowing credential delegation between the entities. Spark uses this technique to periodically refresh and transfer the delegation tokens to the Driver and the Executors, so that they can access secured HDFS, Hive and HBase services.

When you submit a Spark application with `--principal` and `--keytab` options, the first thing Spark Client does is to login via UserGroupInformation class with the provided Kerberos principal and keytab (using its `setupCredentials` [method](https://github.com/cloudera/spark/blob/spark2-2.1.0-cloudera1/yarn/src/main/scala/org/apache/spark/deploy/yarn/Client.scala#L151)).

When preparing [LocalResources](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/yarn/api/records/LocalResource.html) required by the [ContainerLaunchContext](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/yarn/api/records/ContainerLaunchContext.html) ([Client](https://github.com/cloudera/spark/blob/spark2-2.1.0-cloudera1/yarn/src/main/scala/org/apache/spark/deploy/yarn/Client.scala#L389)'s *prepareLocalResources* method), which represents all of the information needed by the NodeManager to launch a container, the Client obtains the delegation tokens for the required services.
 
After obtaining the delegation tokens, the Client also sets three SparkConf configurations, which will be explained in the next sections. The configurations are set with the values below, where *applicationStagingDir* contains the Spark application staging directory, *randomUUID* contains a randomly generated UUID, *nearestTimeOfNextRenewal* contains the minimum token validity time from all the services to which delegation tokens are requested, and *currentTime* contains the current time millis: 

| Key        | Value       |
| ------------- |:-------------:|
| *spark.yarn.credentials.file* | *applicationStagingDir* + "/" + "credentials-" + *randomUUID*
| *spark.yarn.credentials.renewTime* | (*nearestTimeOfNextRenewal* - *currentTime*) * 0.75 + *currentTime*
| *spark.yarn.credentials.updateTime* | (*nearestTimeOfNextRenewal* - *currentTime*) * 0.80 + *currentTime*

*Note:* Since the application was submitted with `--principal` and `--keytab` options, the SparkConf already contains their values in `spark.yarn.principal` and `spark.yarn.keytab` entries. 

Then, the Client adds the obtained delegation tokens to the previously created ContainerLaunchContext, using its `setupSecurityToken` [method](https://github.com/cloudera/spark/blob/spark2-2.1.0-cloudera1/yarn/src/main/scala/org/apache/spark/deploy/yarn/Client.scala#L151).

Finally, the Client creates a [ApplicationSubmissionContext](https://hadoop.apache.org/docs/r2.9.0/api/org/apache/hadoop/yarn/api/records/ApplicationSubmissionContext.html) containing the ContainerLaunchContext, and submits the application using the [YarnClient](http://hadoop.apache.org/docs/r2.9.0/api/org/apache/hadoop/yarn/client/api/YarnClient.html)'s `submitApplication` method.

Now that the application is submitted, it becomes Application Master's responsability to handle the delegation tokens. 

# Application Master's Delegation Token Renewal

When the Application Master starts, an [ApplicationMaster](https://github.com/cloudera/spark/blob/spark2-2.1.0-cloudera1/yarn/src/main/scala/org/apache/spark/deploy/yarn/ApplicationMaster.scala) instance is created. 

First it checks if the `spark.yarn.credentials.file` SparkConf value is set, and if so, it instantiates the [AMCredentialRenewer](https://github.com/cloudera/spark/blob/spark2-2.1.0-cloudera1/yarn/src/main/scala/org/apache/spark/deploy/yarn/security/AMCredentialRenewer.scala) class, which contains all the logic to periodically renew the delegation tokens, and schedules the next credential renewal by invoking AMCredentialRenewer's `scheduleLoginFromKeytab` method.

AMCredentialRenewer reads the `spark.yarn.credentials.renewTime` SparkConf value and assigns it to the `timeOfNextRenewal` variable.

The `scheduleLoginFromKeytab` method simply runs the **renewal scheduling logic** below.

The **renewal scheduling logic** consists in:
1. Determine the `remainingTime` by subtracting `timeOfNextRenewal` and current time millis;
2. Check if the `remainingTime` is greater than 0, if so, schedule the **renewal logic** to be run in `remaningTime` millis. 
    Otherwise, run it immediately.

The **renewal logic** consists in: 
1. Logging-in via UserGroupInformation with the `spark.yarn.principal` and `spark.yarn.keytab` SparkConf values;
2. Obtain the delegation tokens to our secured services by executing [ConfigurableCredentialManager](https://github.com/cloudera/spark/blob/spark2-2.1.0-cloudera1/yarn/src/main/scala/org/apache/spark/deploy/yarn/security/ConfigurableCredentialManager.scala)'s `obtainCredentials` method in a Kerberos context with `ugi.doAs(...)` method;
3. Store the `nearestTimeOfNextRenewal` returned by the previous method, which contains the minimum token validity time;
4. Determine the time of the next credential renewal and the next credential update, using the same formula as the Client:
    * *timeOfNextRenewal* = (*nearestTimeOfNextRenewal* - *currentTime*) * 0.75 + *currentTime*
    * *timeOfNextUpdate* = (*nearestTimeOfNextRenewal* - *currentTime*) * 0.8 + *currentTime*
5. Store the delegation tokens contained within the Credentials object to HDFS, to a path resulting by concatenating the following values:
    * `spark.yarn.credentials.file` SparkConf value
    * "-" delimiter
    * *timeOfNextUpdate*
    * "-" delimiter
    * incremented with each renewal number
    
    E.g. `/user/eventarch/.sparkStaging/application_1547546932241_0041/credentials-fe545216-4a64-47a4-b787-6469ead7ed93-1547576953288-1`
6. Delete credentials files stored in HDFS older than X days, retention period stored in `spark.yarn.credentials.file.retention.days` SparkConf value;
7. Run the **renewal scheduling logic** with the updated `timeOfNextRenewal` value to repeat the cycle.

# Driver and Executor's Credential Update

The required credentials are not acquired by the Driver and the Executors in the same way when they are started. 

By submitting an application in `yarn-client` mode, the Client runs together with the Driver in the same JVM container, so the initial login using the provided `--keytab` and `--principal`, and the delegation token request performed by the Client automatically set for the Driver the required credentials to talk to secured Hadoop services, since both TGT and delegation tokens become available in the Driver's UserGroupInformation. 

Anytime the Executor allocation is requested by the ApplicationMaster, its corresponding ContainerLaunchContext object is populated with the Credentials contained in the ApplicationMaster's UserGroupInformation. ApplicationMaster already contains the delegation tokens when it's started, because the Client passed them in the same way to the ApplicationMaster's ContainerLaunchContext. Every time the ApplicationMaster requests new delegation tokens, it also adds them to its UserGroupInformation, which means that any new allocated Executors will contain delegation tokens.

Both the Driver and the Executor start updating their credentials in the same way when started, using the [CredentialUpdater](https://github.com/cloudera/spark/blob/spark2-2.1.0-cloudera1/yarn/src/main/scala/org/apache/spark/deploy/yarn/security/CredentialUpdater.scala) class.
 
Similarly to how the Application Master renews the credentials, the CredentialUpdater schedules its update task starting with the **update scheduling logic**.
 
The **update scheduling logic** consists in:
1. Determine the `remainingTime` by subtracting `timeOfNextUpdate` and current time millis. `timeOfNextUpdate` equals to the `spark.yarn.credentials.updateTime` SparkConf value when running for the first time;
2. Check if the `remainingTime` is greater than 0, if so, schedule the **update logic** to be run in `remaningTime` millis. 
    Otherwise, schedule the **update logic** to be run in 1 minute. 
 
The **update logic** consists in: 
1. List the HDFS files with a prefix of `spark.yarn.credentials.file` SparkConf value. If there are no matching files, reschedule the next **update logic** to run in 1 minute;
2. Get the most recent file and check if the integer suffix is greater than the last processed. If it's not, reschedule the next **update logic** to run in 1 hour, since the read credentials file is older than expected;
3. Unserialize the stored Credentials object from a file and populate the UserGroupInformation with the delegation tokens stored in it;
4. Update the `timeOfNextUpdate` variable with the value contained in the credentials file name;
5. Run the **update scheduling logic** with the updated `timeOfNextUpdate` value to repeat the cycle.


After the UserGroupInformation of the Driver and the Executors are populated with the delegation tokens, any access to the secured HDFS, Hive or HBase services will succeed.