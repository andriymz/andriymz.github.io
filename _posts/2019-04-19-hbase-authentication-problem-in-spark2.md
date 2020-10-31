---
title:  "HBase Authentication Problem in Spark 2"
excerpt: "This post detailedly explains and presents a workaround solution to a problem with HBase authentication in long-running Spark 2 applications."
classes: wide
categories: [spark]
toc: true
tags: [kerberos, hbase]
author_profile: true
---

Spark uses delegation tokens in all its communications with secure services (e.g. HDFS, Hive, HBase), however there is a problem in refreshing HBase tokens in versions 2.0.x, 2.1.x and 2.2.x of Spark in `yarn-client` mode, making any long-running applications unable to access HBase after a period of 7 days, default period during which the initially obtained authentication token can be renewed. In this blog post I'm addressing this issue by explaining the root cause of this problem and presenting my workaround solution.

To fully understand how Spark uses delegation tokens for secure communication in hadoop clusters you can check my [post](/spark/how-spark-uses-kerberos-authentication) about Spark and Kerberos authentication.


# When delegation tokens are requested?

Delegation tokens are requested initially by the Client before submitting the application to YARN. Since in `yarn-client` deploy mode the Client and the Driver run in the same JVM container, you can check the tokens your application requests in the Driver's logs:

```text
2019-01-19 08:45:30,761 INFO org.apache.spark.deploy.yarn.security.HDFSCredentialProvider: getting token for namenode: hdfs://xp-sonae1.xpand.com:8020/user/eventarch
2019-01-19 08:45:30,854 INFO org.apache.hadoop.hdfs.DFSClient: Created token for eventarch: HDFS_DELEGATION_TOKEN owner=eventarch/xp-sonae3.xpand.com@XPAND.COM, renewer=yarn, realUser=, issueDate=1547905530830,
maxDate=1547905830830, sequenceNumber=7796, masterKeyId=394 on 10.0.0.4:8020
2019-01-19 08:45:30,862 INFO org.apache.hadoop.hdfs.DFSClient: Created token for eventarch: HDFS_DELEGATION_TOKEN owner=eventarch/xp-sonae3.xpand.com@XPAND.COM, renewer=eventarch, realUser=, issueDate=1547905530
878, maxDate=1547905830878, sequenceNumber=7797, masterKeyId=394 on 10.0.0.4:8020
 
2019-01-19 08:45:32,929 INFO org.apache.spark.deploy.yarn.security.HBaseCredentialProvider: Get token from HBase: Kind: HBASE_AUTH_TOKEN, Service: 410baf3c-6ca8-4814-b2af-ad67caf17e6f, Ident: (org.apache.hadoop.
hbase.security.token.AuthenticationTokenIdentifier@3)
 
2019-01-19 08:45:33,968 INFO org.apache.spark.deploy.yarn.security.HiveCredentialProvider: Get Token from hive metastore: Kind: HIVE_DELEGATION_TOKEN, Service: , Ident: 00 27 65 76 65 6e 74 61 72 63 68 2f 78 70 2d 73 6f 6e 61 65 33 2e 78 70 61 6e 64 2e 63 6f 6d 40 58 50 41 4e 44 2e 43 4f 4d 04 68 69 76 65 00 8a 01 68 66 5c 07 eb 8a 01 68 8a 68 8b eb 13 01
2019-01-19 08:45:33,971 INFO hive.metastore: Closed a connection to metastore, current connections: 0
```

After the application is submitted to YARN, it becomes Application Master's responsability to refresh the tokens.

# How Application Master requests the tokens?

Even though the delegation tokens can be renewed during the token's lifespan, Application Master simply requests new ones from the secured services upon reaching 75% of the minimal token validity.

Before requesting the tokens, Application Master logs into the KDC with the credentials provided through spark-submit `--principal` and `--keytab` arguments. To give Application Master access to the submitted *keytab*, Spark uploads it into the application's staging directory and sets up the Kerberos principal and keytab filename through `spark.yarn.principal` and `spark.yarn.keytab` SparkConf configurations, also accessible by the Application Master.

```text
2019-01-19 08:45:33,998 INFO org.apache.spark.deploy.yarn.Client: To enable the Application Master to login from keytab, credentials are being copied over to the Application Master via the YARN Secure Distributed Cache.
2019-01-19 08:45:34,001 INFO org.apache.spark.deploy.yarn.Client: Uploading resource file:/var/run/cloudera-scm-agent/process/1649-event_architecture-EVENT_ARCHITECTURE_DISPATCHER_RFID/event_architecture.keytab -> hdfs://xp-sonae1.xpand.com:8020/user/eventarch/.sparkStaging/application_1547896472312_0007/event_architecture.keytab
```

If you submit a Spark 2 application in `yarn-client` mode, you can check through its logs that the Application Master only requests delegation tokens for HDFS and Hive services (no token for HBase):

```text
2019/01/15 13:25:12 [INFO] [security.AMCredentialRenewer] Attempting to login to KDC using principal: eventarch/xp-sonae3.xpand.com@XPAND.COM
2019/01/15 13:25:12 [INFO] [security.AMCredentialRenewer] Successfully logged into KDC.
 
2019/01/15 13:25:12 [INFO] [security.HDFSCredentialProvider] getting token for namenode: hdfs://xp-sonae1.xpand.com:8020/user/eventarch
2019/01/15 13:25:12 [INFO] [hdfs.DFSClient] Created token for eventarch: HDFS_DELEGATION_TOKEN owner=eventarch/xp-sonae3.xpand.com@XPAND.COM, renewer=yarn, realUser=, issueDate=1547576712738, maxDate=1547577012738, sequenceNumber=6476, masterKeyId=382 on 10.0.0.4:8020
2019/01/15 13:25:12 [INFO] [hdfs.DFSClient] Created token for eventarch: HDFS_DELEGATION_TOKEN owner=eventarch/xp-sonae3.xpand.com@XPAND.COM, renewer=eventarch, realUser=, issueDate=1547576712767, maxDate=1547577012767, sequenceNumber=6477, masterKeyId=382 on 10.0.0.4:8020
 
2019/01/15 13:25:14 [INFO] [hive.metastore] Trying to connect to metastore with URI thrift://xp-sonae2.xpand.com:9083
2019/01/15 13:25:14 [INFO] [hive.metastore] Opened a connection to metastore, current connections: 1
2019/01/15 13:25:14 [INFO] [hive.metastore] Connected to metastore.
2019/01/15 13:25:15 [INFO] [metadata.Hive] Registering function mymyupper org.hue.udf.MyUpper
2019/01/15 13:25:15 [INFO] [security.HiveCredentialProvider] Get Token from hive metastore: Kind: HIVE_DELEGATION_TOKEN, Service: , Ident: 00 27 65 76 65 6e 74 61 72 63 68 2f 78 70 2d 73 6f 6e 61 65 33 2e 78 70 61 6e 64 2e 63 6f 6d 40 58 50 41 4e 44 2e 43 4f 4d 04 68 69 76 65 00 8a 01 68 52 c2 a8 cd 8a 01 68 76 cf 2c cd 8f 96 01
2019/01/15 13:25:15 [INFO] [hive.metastore] Closed a connection to metastore, current connections: 0
```

Requested delegation tokens are written to HDFS, making them accessible by the Driver and Executor instances:

```text
2019/01/15 13:25:15 [INFO] [security.AMCredentialRenewer] Writing out delegation tokens to hdfs://xp-sonae1.xpand.com:8020/user/eventarch/.sparkStaging/application_1547546932241_0041/credentials-fe545216-4a64-47a4-b787-6469ead7ed93-1547576953288-1.tmp
2019/01/15 13:25:15 [INFO] [security.AMCredentialRenewer] Delegation Tokens written out successfully. Renaming file to hdfs://xp-sonae1.xpand.com:8020/user/eventarch/.sparkStaging/application_1547546932241_0041/credentials-fe545216-4a64-47a4-b787-6469ead7ed93-1547576953288-1
2019/01/15 13:25:15 [INFO] [security.AMCredentialRenewer] Delegation token file rename complete.
2019/01/15 13:25:15 [INFO] [security.AMCredentialRenewer] Scheduling login from keytab in 222651 millis.
```

# HBase token renewal problem

HBase tokens are not refreshed by the Application Master because the required HBase classes are not present in Application Master container's classpath, and there is no way to manually provide it in the scope of a single application. This was tricky to catch, because the default Application Master's logging level didn't show the DEBUG exception logs present in [HBaseCredentialProvider](https://github.com/cloudera/spark/blob/spark2-2.1.0-cloudera1/yarn/src/main/scala/org/apache/spark/deploy/yarn/security/HBaseCredentialProvider.scala). Only after changing the logging level to include the DEBUG or the code itself you will be able to confirm the error cause.

```text
2019/01/15 13:25:14 [DEBUG] [security.HBaseCredentialProvider] Failed to get token from service hbase
java.lang.ClassNotFoundException: org.apache.hadoop.hbase.security.token.TokenUtil
    at java.net.URLClassLoader.findClass(URLClassLoader.java:381)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
    at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:331)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
```

Since the HBase token can't be renewed, the Application Master always writes to HDFS the same initially obtained by the Client HBase token, together with the successfully refreshed HDFS and Hive ones. When Driver and Executors update their credentials with the tokens contained in the file read from the HDFS, they only refresh the HDFS and Hive tokens. If the application runs for more than 7 days, the default period during which a HBase authentication token can be renewed, any request to secured HBase will fail.

**Jira Issues**

The [SPARK-21377](https://issues.apache.org/jira/browse/SPARK-21377) Jira issue reports this problem and apparently the problem was fixed in Spark 2.3. Another relevant issue regarding delegation token renewal is [SPARK-25689](https://issues.apache.org/jira/browse/SPARK-25689), which was also resolved and points that since Spark 3.0 the Application Master is no longer responsible for the token renewal in `yarn-client` mode.


# Workaround Solution

To overcome this problem I've found a way to manually request the HBase authentication token on the Driver side, read the Application Master generated credentials file, replace the invalid HBase token and write back the credentials file, making the underlying Spark implementation read my updated credentials file instead.

**Steps**

1. Read the credentials file from hdfs://xp-sonae1.xpand.com:8020/user/eventarch/.sparkStaging/application_1547896472312_0015/**credentials-c820302c-bc91-4a14-84a0-f1b92caf7a4c-1547921890628-1**
2. Request the HBase token using the [TokenUtil](https://github.com/apache/hbase/blob/master/hbase-server/src/main/java/org/apache/hadoop/hbase/security/token/TokenUtil.java) class;
3. Replace the HBase token in the unserialized [Credentials](https://hadoop.apache.org/docs/r2.8.3/api/org/apache/hadoop/security/Credentials.html) instance;
4. Store the updated Credentials instance into HDFS under hdfs://xp-sonae1.xpand.com:8020/user/eventarch/.sparkStaging/application_1547896472312_0015/**credentials-c820302c-bc91-4a14-84a0-f1b92caf7a4c-1547921890628-10001**


Application Master renews the delegation tokens every 75% of `Min(service provider token validity time)`, but the update from the HDFS by the Driver and Executor instances occurs every 80% of this time. This means we have a 5% time window to read the fresh delegation tokens the Application Master stored in HDFS and apply our logic.

Considering the default token validity of 24h, Spark will renew the tokens from the services every `75% * 24h = 18h` and update them slightly later in `5% * 24h = 1.2h`.
The proposed solution takes this into consideration and performs the merging logic to be done 2% * 24h before the credentials are updated by the Driver and the Executors.

To apply this workaround on your project you just need to include the provided `ManualDelegationTokenRenewal` class below and invoke its `startCredentialUpdater()` method in your Driver code. A thread will be spawned to automagically update the credentials file and fix the HBase access problem for long-running applications.

You can check the [CredentialUpdater](https://github.com/cloudera/spark/blob/spark2-2.1.0-cloudera1/yarn/src/main/scala/org/apache/spark/deploy/yarn/security/CredentialUpdater.scala)'s logs to confirm that the updated credentials file is read from the HDFS in the Driver and Executor logs.

```text
2019-01-19 13:14:24,498 INFO org.apache.spark.deploy.yarn.security.CredentialUpdater: Reading new credentials from hdfs://xp-sonae1.xpand.com:8020/user/eventarch/.sparkStaging/application_1547896472312_0015/credentials-c820302c-bc91-4a14-84a0-f1b92caf7a4c-1547921890628-100001
2019-01-19 13:18:10,633 INFO org.apache.spark.deploy.yarn.security.CredentialUpdater: Reading new credentials from hdfs://xp-sonae1.xpand.com:8020/user/eventarch/.sparkStaging/application_1547896472312_0015/credentials-c820302c-bc91-4a14-84a0-f1b92caf7a4c-1547922115956-100002
2019-01-19 13:21:55,960 INFO org.apache.spark.deploy.yarn.security.CredentialUpdater: Reading new credentials from hdfs://xp-sonae1.xpand.com:8020/user/eventarch/.sparkStaging/application_1547896472312_0015/credentials-c820302c-bc91-4a14-84a0-f1b92caf7a4c-1547922341219-100003
2019-01-19 13:25:41,223 INFO org.apache.spark.deploy.yarn.security.CredentialUpdater: Reading new credentials from hdfs://xp-sonae1.xpand.com:8020/user/eventarch/.sparkStaging/application_1547896472312_0015/credentials-c820302c-bc91-4a14-84a0-f1b92caf7a4c-1547922566404-100004
2019-01-19 13:29:26,408 INFO org.apache.spark.deploy.yarn.security.CredentialUpdater: Reading new credentials from hdfs://xp-sonae1.xpand.com:8020/user/eventarch/.sparkStaging/application_1547896472312_0015/credentials-c820302c-bc91-4a14-84a0-f1b92caf7a4c-1547922791699-100005
2019-01-19 13:33:11,702 INFO org.apache.spark.deploy.yarn.security.CredentialUpdater: Reading new credentials from hdfs://xp-sonae1.xpand.com:8020/user/eventarch/.sparkStaging/application_1547896472312_0015/credentials-c820302c-bc91-4a14-84a0-f1b92caf7a4c-1547923016903-100006
2019-01-19 13:36:56,907 INFO org.apache.spark.deploy.yarn.security.CredentialUpdater: Reading new credentials from hdfs://xp-sonae1.xpand.com:8020/user/eventarch/.sparkStaging/application_1547896472312_0015/credentials-c820302c-bc91-4a14-84a0-f1b92caf7a4c-1547923242083-100007
2019-01-19 13:40:42,087 INFO org.apache.spark.deploy.yarn.security.CredentialUpdater: Reading new credentials from hdfs://xp-sonae1.xpand.com:8020/user/eventarch/.sparkStaging/application_1547896472312_0015/credentials-c820302c-bc91-4a14-84a0-f1b92caf7a4c-1547923467255-100008
2019-01-19 13:44:27,258 INFO org.apache.spark.deploy.yarn.security.CredentialUpdater: Reading new credentials from hdfs://xp-sonae1.xpand.com:8020/user/eventarch/.sparkStaging/application_1547896472312_0015/credentials-c820302c-bc91-4a14-84a0-f1b92caf7a4c-1547923692432-100009
2019-01-19 13:48:12,436 INFO org.apache.spark.deploy.yarn.security.CredentialUpdater: Reading new credentials from hdfs://xp-sonae1.xpand.com:8020/user/eventarch/.sparkStaging/application_1547896472312_0015/credentials-c820302c-bc91-4a14-84a0-f1b92caf7a4c-1547923917603-100010
```

Full update iteration retrieved from the Driver's looks like this:

```text
2019-01-19 13:10:41,744 INFO org.apache.spark.deploy.yarn.security.CredentialUpdater: Scheduling credentials refresh from HDFS in 222744 ms.
2019-01-19 13:10:45,950 INFO com.xpandit.bdu.sonae.ea.commons.spark.ManualDelegationTokenRenewal: Scheduling credential merge in 214167ms
2019-01-19 13:14:20,120 INFO com.xpandit.bdu.sonae.ea.commons.spark.ManualDelegationTokenRenewal: Merging credentials
2019-01-19 13:14:20,140 INFO com.xpandit.bdu.sonae.ea.commons.spark.ManualDelegationTokenRenewal: Reading new credentials from hdfs://xp-sonae1.xpand.com:8020/user/eventarch/.sparkStaging/application_1547896472312_0015/credentials-c820302c-bc91-4a14-84a0-f1b92caf7a4c-1547921890628-1
2019-01-19 13:14:20,166 INFO com.xpandit.bdu.sonae.ea.commons.spark.ManualDelegationTokenRenewal: Logged as: eventarch/xp-sonae3.xpand.com@XPAND.COM (auth:KERBEROS)
2019-01-19 13:14:20,259 INFO com.xpandit.bdu.sonae.ea.commons.spark.ManualDelegationTokenRenewal: Got HBase token: org.apache.hadoop.hbase.security.token.AuthenticationTokenIdentifier@44
2019-01-19 13:14:20,262 INFO com.xpandit.bdu.sonae.ea.commons.spark.ManualDelegationTokenRenewal: Writing out delegation tokens to hdfs://xp-sonae1.xpand.com:8020/user/eventarch/.sparkStaging/application_1547896472312_0015/credentials-c820302c-bc91-4a14-84a0-f1b92caf7a4c-1547921890628-100001.tmp
2019-01-19 13:14:20,293 INFO com.xpandit.bdu.sonae.ea.commons.spark.ManualDelegationTokenRenewal: Delegation Tokens written out successfully. Renaming file to hdfs://xp-sonae1.xpand.com:8020/user/eventarch/.sparkStaging/application_1547896472312_0015/credentials-c820302c-bc91-4a14-84a0-f1b92caf7a4c-1547921890628-100001
2019-01-19 13:14:20,305 INFO com.xpandit.bdu.sonae.ea.commons.spark.ManualDelegationTokenRenewal: Delegation token file rename complete.
2019-01-19 13:14:20,306 INFO com.xpandit.bdu.sonae.ea.commons.spark.ManualDelegationTokenRenewal: Scheduling credential merge in 225716ms
2019-01-19 13:14:24,498 INFO org.apache.spark.deploy.yarn.security.CredentialUpdater: Reading new credentials from hdfs://xp-sonae1.xpand.com:8020/user/eventarch/.sparkStaging/application_1547896472312_0015/credentials-c820302c-bc91-4a14-84a0-f1b92caf7a4c-1547921890628-100001
``` 

**ManualDelegationTokenRenewal class**

```scala
package com.xpandit.bdu.sonae.ea.commons.spark
 
import java.security.PrivilegedExceptionAction
import java.util.concurrent.{Executors, TimeUnit}
 
import com.xpandit.bdu.commons.core.Loggable
import org.apache.hadoop.conf.Configuration
import org.apache.hadoop.fs.{FileSystem, Path}
import org.apache.hadoop.hbase.HBaseConfiguration
import org.apache.hadoop.hbase.client.{Connection, ConnectionFactory}
import org.apache.hadoop.hbase.security.token.TokenUtil
import org.apache.hadoop.security.token.{Token, TokenIdentifier}
import org.apache.hadoop.security.{Credentials, UserGroupInformation}
import org.apache.spark.SparkConf
import org.apache.spark.deploy.SparkHadoopUtil
 
/**
  * @author Andriy Z.
  */
object ManualDelegationTokenRenewal extends Loggable {
 
  private val scheduledExecutor = Executors.newSingleThreadScheduledExecutor()
 
  private val sparkHadoopUtil = new SparkHadoopUtil()
  private val hadoopConf = new Configuration()
 
  private var lastCredentialsFileSuffix = 100000 //Why not
 
  private val CREDENTIALS_FILE_PATH_CONF = "spark.yarn.credentials.file"
  private val CREDENTIALS_UPDATE_TIME_CONF = "spark.yarn.credentials.updateTime"
 
  private val KERBEROS_PRINCIPAL_CONF = "spark.yarn.principal"
  private val KERBEROS_KEYTAB_CONF = "spark.yarn.keytab"
 
  //Spark renews the delegation tokens every 75% of min(service provider token validity time)
  //But the update from the HDFS happens every 80% of this time.
  //This means we have a 5% time window to read the fresh delegation tokens the Spark stored in HDFS,
  //apply our logic and write our merged credentials file to HDFS, so that the Executors and the Driver read our credentials
  //with valid HBase authentication token.
  //This ratio parameter ensures that the next credential merge happens within this time frame.
  //E.g: Considering the default token validity of 24h
  //Spark will renew the tokens from the services every 75% * 24h = 18h and update them slightly later 5% * 24h = 1.2h
  //Considering our ratio of 2% we would merge the credentials ~28min (2% of 24h) before the Spark update.
  private val MERGE_CREDENTIALS_BEFORE_SPARK_UPDATE_RATIO = 0.98
 
  private var sparkConf: SparkConf = _
 
  def startCredentialUpdater(sparkConf: SparkConf) = {
    this.sparkConf = sparkConf
 
    //When the delegation tokens are requested for the first time by the Client, no credentials file is written to HDFS.
    //Since the HBase authentication token is correctly retrieved for the first time, the Driver and the Executors have
    //valid credentials until the token expires, because the Application Master no longer requests HBase tokens.
    //Since the Client stores the next credentials update time in spark.yarn.credentials.updateTime,
    //We need to simply schedule our credentials merge to start slightly before.
 
    val nextUpdateTime = sparkConf.getTimeAsMs(CREDENTIALS_UPDATE_TIME_CONF)
    val updatePeriodicity = nextUpdateTime - System.currentTimeMillis()
    val nextCredentialMergeRemainingTime = (updatePeriodicity * MERGE_CREDENTIALS_BEFORE_SPARK_UPDATE_RATIO).toLong
 
    INFO(s"Scheduling credential merge in ${nextCredentialMergeRemainingTime}ms")
    scheduledExecutor.schedule(new Runnable {
      override def run(): Unit = logUncaughtExceptions(mergeCredentials(sparkConf))
    }, nextCredentialMergeRemainingTime, TimeUnit.MILLISECONDS)
  }
 
  private def mergeCredentials(sparkConf: SparkConf): Unit = {
    INFO("Merging credentials")
 
    val credentialsFilePath = new Path(sparkConf.get(CREDENTIALS_FILE_PATH_CONF))
    val remoteFs = FileSystem.get(hadoopConf)
 
    SparkHadoopUtil.get.listFilesSorted(
      remoteFs, credentialsFilePath.getParent,
      credentialsFilePath.getName, SparkHadoopUtil.SPARK_YARN_CREDS_TEMP_EXTENSION)
      .lastOption
      .map { credentialsStatus =>
        INFO("Reading new credentials from " + credentialsStatus.getPath)
        val newCredentials = getCredentialsFromHDFSFile(remoteFs, credentialsStatus.getPath)
        lastCredentialsFileSuffix += 1
 
        val hbaseToken = obtainHBaseToken()
        if(hbaseToken != null) {
          INFO("Got HBase token: " + hbaseToken.decodeIdentifier().toString)
          newCredentials.addToken(hbaseToken.getService, hbaseToken)
        }
 
        writeMergedCredentialsToHDFS(newCredentials, remoteFs, credentialsFilePath.getParent, credentialsStatus.getPath.getName)
 
        val nextUpdateTime = getTimeOfNextUpdateFromFileName(credentialsStatus.getPath)
        val updatePeriodicity = nextUpdateTime - System.currentTimeMillis()
        val nextCredentialMergeRemainingTime = (updatePeriodicity * MERGE_CREDENTIALS_BEFORE_SPARK_UPDATE_RATIO).toLong
 
        INFO(s"nextUpdateTime: $nextUpdateTime")
        INFO(s"nextUpdateRemainingTime: $updatePeriodicity")
 
        INFO(s"Scheduling credential merge in ${nextCredentialMergeRemainingTime}ms")
        scheduledExecutor.schedule(new Runnable {
          override def run(): Unit = logUncaughtExceptions(mergeCredentials(sparkConf))
        }, nextCredentialMergeRemainingTime, TimeUnit.MILLISECONDS)
      }.getOrElse {
      // This should not happen, since the Client
      INFO(s"No new credential file found, trying again in a minute.")
 
      scheduledExecutor.schedule(new Runnable {
        override def run(): Unit = logUncaughtExceptions(mergeCredentials(sparkConf))
      }, 1, TimeUnit.MINUTES)
    }
  }
 
  /**
    * Requests HBase new delegation / authentication token from the service
    */
  def obtainHBaseToken(): Token[_ <: TokenIdentifier] = {
    val principal = sparkConf.get(KERBEROS_PRINCIPAL_CONF)
    val keytab = sparkConf.get(KERBEROS_KEYTAB_CONF)
 
    INFO(s"Principal: $principal")
    INFO(s"Keytab: $keytab")
 
    val hbaseConf = HBaseConfiguration.create(hadoopConf)
    var hbaseConn: Connection = null
    var hbaseToken: Token[_ <: TokenIdentifier] = null
 
    try {
      val ugi = UserGroupInformation.loginUserFromKeytabAndReturnUGI(principal, keytab)
      INFO("Logged as: " + ugi.toString)
 
      hbaseToken = ugi.doAs(new PrivilegedExceptionAction[Token[_ <: TokenIdentifier]] {
        override def run(): Token[_ <: TokenIdentifier] = {
          hbaseConn = ConnectionFactory.createConnection(hbaseConf)
          TokenUtil.obtainToken(hbaseConn)
        }
      })
      ugi.logoutUserFromKeytab()
    }
    catch {
      case t: Exception =>
        ERROR(t)(s"Couldn't request the HBase authentication token")
    }
    finally {
      if(hbaseConn != null) hbaseConn.close()
    }
 
    hbaseToken
  }
 
  /**
    * Reads org.apache.hadoop.security.Credentials stored in HDFS
    */
  private def getCredentialsFromHDFSFile(remoteFs: FileSystem, tokenPath: Path): Credentials = {
    val stream = remoteFs.open(tokenPath)
    try {
      val newCredentials = new Credentials()
      newCredentials.readTokenStorageStream(stream)
      newCredentials
    } finally {
      stream.close()
    }
  }
 
  /**
    * Stores the merged org.apache.hadoop.security.Credentials into the HDFS
    */
  private def writeMergedCredentialsToHDFS(newCredentials: Credentials, remoteFs: FileSystem,
                                           credentialsFilePath: Path, sparkGeneratedCredentialsFilename: String) = {
    val newFilename = replaceSuffixForCredentialsPath(sparkGeneratedCredentialsFilename, lastCredentialsFileSuffix)
    val finalCredentialsPath = new Path(credentialsFilePath, newFilename)
    val tempCredentialsPath = new Path(credentialsFilePath, newFilename + SparkHadoopUtil.SPARK_YARN_CREDS_TEMP_EXTENSION)
 
    INFO(s"Writing out delegation tokens to $tempCredentialsPath")
    newCredentials.writeTokenStorageFile(tempCredentialsPath, hadoopConf)
    INFO(s"Delegation Tokens written out successfully. Renaming file to $finalCredentialsPath")
    remoteFs.rename(tempCredentialsPath, finalCredentialsPath)
    INFO("Delegation token file rename complete.")
  }
 
 
  /**
    * Retrieve the next credentials update time from the credentials file name
    * For filename: credentials-90e1d3f6-99bc-4ec2-8531-ebf8f809b321-1547643753335-1
    * Result: 1547643753335
    */
  private def getTimeOfNextUpdateFromFileName(credentialsPath: Path): Long = {
    val name = credentialsPath.getName
    val index = name.lastIndexOf(SparkHadoopUtil.SPARK_YARN_CREDS_COUNTER_DELIM)
    val slice = name.substring(0, index)
    val last2index = slice.lastIndexOf(SparkHadoopUtil.SPARK_YARN_CREDS_COUNTER_DELIM)
    name.substring(last2index + 1, index).toLong
  }
 
  /**
    * For filename: credentials-90e1d3f6-99bc-4ec2-8531-ebf8f809b321-1547643753335-1
    * and newSuffix: 99999
    * Replaces the suffix with the provided value
    * Result: credentials-90e1d3f6-99bc-4ec2-8531-ebf8f809b321-1547643753335-99999
    */
  def replaceSuffixForCredentialsPath(filename: String, newSuffix: Int): String = {
    filename.substring(0, filename.lastIndexOf(SparkHadoopUtil.SPARK_YARN_CREDS_COUNTER_DELIM) + 1) + newSuffix
  }
 
  /**
    * Execute the given block, logging and re-throwing any uncaught exception.
    * This is particularly useful for wrapping code that runs in a thread, to ensure
    * that exceptions are printed, and to avoid having to catch Throwable.
    */
  private def logUncaughtExceptions[T](f: => T): T = {
    try {
      f
    } catch {
      case t: Throwable =>
        ERROR(t)(s"Uncaught exception in thread ${Thread.currentThread().getName}")
        throw t
    }
  }
}
```
