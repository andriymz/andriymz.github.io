---
title:  "Authentication using Kerberos"
excerpt: "In this post you will see how Kerberos authentication with pure *Java Authentication and Authorization Service (JAAS)* works and how to use the *UserGroupInformation* class for each of its authentication features, such as logging-in from ticket cache or keytab, TGT renewal, impersonation with proxy-users and delegation tokens."
classes: wide
categories: [kerberos]
toc: true
tags: [security, jaas]
author_profile: true
---
 
In this post you will see how Kerberos authentication with pure *Java Authentication and Authorization Service (JAAS)* works and how to use the *UserGroupInformation* class for each of its authentication features, such as logging-in from ticket cache or keytab, TGT renewal, impersonation with proxy-users and delegation tokens.
 
I assume that you are already familiar with Kerberos components and terms, such as *Key Distribution Center (KDC)*, *Ticket-Granting Ticket (TGT)*, *Ticket-Granting Service (TGS)*, etc.
 
# Authenticate with JAAS configuration and a keytab

*Java Authentication and Authorization Service (JAAS)* is the Java implementation of the standard *Pluggable Authentication Module (PAM)*, allowing applications to be independent from underlying authentication technologies.

The client only interacts with the [LoginContext](https://docs.oracle.com/javase/7/docs/api/javax/security/auth/login/LoginContext.html) object, which reads the [Configuration](https://docs.oracle.com/javase/7/docs/api/javax/security/auth/login/Configuration.html) and instantiates the specified in it [LoginModules](https://docs.oracle.com/javase/7/docs/api/javax/security/auth/spi/LoginModule.html). As a result, different LoginModules can be plugged in under an application without any modifications to the application's code itself.


A login Configuration file consists or one or more entries specifying which authentication technology should be used for an application. The structure of each entry is the following:

```text
Identifier {
    ModuleClass  Flag    ModuleOptions;
    ModuleClass  Flag    ModuleOptions;
    ModuleClass  Flag    ModuleOptions;
};
```

*Identifier* - the name that the application implementing the JAAS authentication should use to refer to this entry.

*ModuleClass* - specifies the fully qualified class name for a class that implements a particular authentication technology.

*Flag* - the flag value indicates whether success of the LoginModule is "required", "requisite", "sufficient", or "optional".

*ModuleOptions* - space-separated list of values which are passed directly to the underlying LoginModule.

## Krb5LoginModule

[Krb5LoginModule](https://docs.oracle.com/javase/8/docs/jre/api/security/jaas/spec/com/sun/security/auth/module/Krb5LoginModule.html) uses Kerberos as the underlying authentication technology.

In order to use this login module the JAAS configuration file should have the following structure:
```text
JaasSample {
  com.sun.security.auth.module.Krb5LoginModule required
  useTicketCache=false
  useKeyTab=true
  doNotPrompt=true
  keyTab="/tmp/path/sparker.keytab"
  principal="sparker@DOMAIN";
};
```

The table below presents some of the possible and invalid option combinations:

| Options        | Result       |
| ------------- |:-------------:|
| doNotPrompt=true  |   This is an illegal combination since none of useTicketCache, useKeyTab, useFirstPass and tryFirstPass is set and the user can not be prompted for the password |
| ticketCache=_filename_ |   This is an illegal combination since useTicketCache is not set to true and the ticketCache is set. A configuration error will occur.       |
| renewTGT=true | This is an illegal combination since useTicketCache is not set to true and renewTGT is set. A configuration error will occur.       |
| storeKey=true<br>useTicketCache=true<br>doNotPrompt=true |  This is an illegal combination since storeKey is set to true but the key can not be obtained either by prompting the user or from the keytab, or from the shared state. A configuration error will occur. |
| keyTab=_filename_ <br>doNotPrompt=true | This is an illegal combination since useKeyTab is not set to true and the keyTab is set. A configuration error will occur.  |
| debug=true  | Prompt the user for the principal name and the password. Use the authentication exchange to get TGT from the KDC and populate the Subject with the principal and TGT. Output debug messages.  |
| useTicketCache=true<br>doNotPrompt=true  | Check the default cache for TGT and populate the Subject with the principal and TGT. If the TGT is not available, do not prompt the user, instead fail the authentication.  |
| principal=_name_ <br> useTicketCache=true<br>doNotPrompt=true  | Get the TGT from the default cache for the principal and populate the Subject's principal and private creds set. If ticket cache is not available or does not contain the principal's TGT authentication will fail. |
| useTicketCache=true <br> ticketCache=_filename_ <br> useKeyTab=true <br> keyTab=_filepath_ <br> principal=_principal_ <br> doNotPrompt=true | Search the cache for the principal's TGT. If it is not available use the key in the keytab to perform authentication exchange with the KDC and acquire the TGT. The Subject will be populated with the principal and the TGT. If the key is not available or valid then authentication will fail. |
| useTicketCache=true <br> ticketCache=_filename_ <br> | The TGT will be obtained from the cache specified. The Kerberos principal name used will be the principal name in the Ticket cache. If the TGT is not available in the ticket cache the user will be prompted for the principal name and the password. The TGT will be obtained using the authentication exchange with the KDC. The Subject will be populated with the TGT.  |
| useKeyTab=true <br> keyTab=_filepath_ <br> principal=_name_ <br> storeKey=true | The key for the principal will be retrieved from the keytab. If the key is not available in the keytab the user will be prompted for the principal's password. The Subject will be populated with the principal's key either from the keytab or derived from the password entered. |
| useKeyTab=true <br> keyTab=_filepath_ <br> storeKey=true <br> doNotPrompt=false | The user will be prompted for the service principal name. If the principal's longterm key is available in the keytab , it will be added to the Subject's private credentials. An authentication exchange will be attempted with the principal name and the key from the Keytab. If successful the TGT will be added to the Subject's private credentials set. Otherwise the authentication will fail. |
| isInitiator=false <br> useKeyTab=true <br> keyTab=_filepath_ <br> storeKey=true <br> principal=* | The acceptor will be an unbound acceptor and it can act as any principal as long that principal has keys in the keytab. |
| useTicketCache=true <br> ticketCache=_filepath_ <br> useKeyTab=true <br> keyTab=_filepath_ <br> storeKey=true <br> principal=_name_  | The client's TGT will be retrieved from the ticket cache and added to the Subject's private credentials. If the TGT is not available in the ticket cache, or the TGT's client name does not match the principal name, Java will use a secret key to obtain the TGT using the authentication exchange and added to the Subject's private credentials. This secret key will be first retrieved from the keytab. If the key is not available, the user will be prompted for the password. In either case, the key derived from the password will be added to the Subject's private credentials set.  |
| isInitiator=false | Configured to act as acceptor only, credentials are not acquired via AS exchange. For acceptors only, set this value to false. For initiators, do not set this value to false.  |
| isInitiator=true| Configured to act as initiator, credentials are acquired via AS exchange. For initiators, set this value to true, or leave this option unset, in which case default value (true) will be used. |


## Important classes/interfaces

The JAAS framework is implemented by many classes/interfaces, but I'll try to summarize the main ones in the table below: 

| Class/Interface        | Purpose       |
| ------------- |:-------------:|
| [LoginContext](https://docs.oracle.com/javase/7/docs/api/javax/security/auth/login/LoginContext.html)    | Entry authentication point to be used by the clients. <br> Contains the basic *login(), logout()* and *getSubject()* methods used to authenticate a subject. <br> Provides a way of creating a secure context from any implementation following the [LoginModule](https://docs.oracle.com/javase/7/docs/api/javax/security/auth/spi/LoginModule.html) interface.
| [Subject](https://docs.oracle.com/javase/7/docs/api/javax/security/auth/Subject.html) | Contains all the information of an authenticated entity (person or service). <br> Once authenticated, a Subject is populated with associated identities called [Principals](https://docs.oracle.com/javase/8/docs/api/java/security/Principal.html) <br> In addiction, a Subject also owns security-related attributes, more specifically a set of public and private credential objects. 
| [Principal](https://docs.oracle.com/javase/8/docs/api/java/security/Principal.html)  | Interface representing the abstract notion of a principal used to represent an authenticated entity.  |
| [KerberosPrincipal](https://docs.oracle.com/javase/7/docs/api/javax/security/auth/kerberos/KerberosPrincipal.html) | Class instance populated in the Subject when Krb5LoginModule is used. <br> Stores the Kerberos principal and realm. |
| [KerberosTicket](https://docs.oracle.com/javase/7/docs/api/javax/security/auth/kerberos/KerberosTicket.html) | Encapsulates a Kerberos ticket and associated information sent by the KDC to a client or recovered from the ticket cache. <br> Its instance is stored in the private credentials set of a Subject.  |
| [CallbackHandler](https://docs.oracle.com/javase/7/docs/api/javax/security/auth/callback/CallbackHandler.html) |  An application implements a CallbackHandler and passes it to underlying security services so that they may interact with the application to retrieve specific authentication data, such as usernames and passwords, or to display certain information, such as error and warning messages.  |

## Authenticate by providing JAAS configuration at application start

A common way of passing the JAAS configuration to be used is setting the `-Djava.security.auth.login.config` Java option with the configuration file path.

Then in your application you should first create a LoginContext instance, by providing the identifier of the JAAS configuration entry and the CallBackHandler. 

The provided [TextCallbackHandler](https://docs.oracle.com/javase/8/docs/jre/api/security/jaas/spec/com/sun/security/auth/callback/TextCallbackHandler.html) simply prompts and reads from the command line for answers to authentication questions.

Once the LoginContext instance is created we call its *login* method, which calls methods in the Krb5LoginModule to perform the login and authentication. The Krb5LoginModule will utilize the TextCallbackHandler to obtain the user name and password. Then the Krb5LoginModule will use this information to get the user credentials from the Kerberos KDC.

The Krb5LoginModule populates the Subject contained within the LoginContext class with a Kerberos Principal representing the user and the user's credentials (TGT) in Subject's private credential set, as mentioned previously.

```java
LoginContext lc = new LoginContext("JaasSample", new TextCallbackHandler());
lc.login()
```

Then we can retrieve the authenticated Subject instance by calling the LoginContext's *getSubject* method.

Finally, using the Subject's *doAs* static method you now should be able to access the Kerberos secured services.

```java
Subject subject = lc.getSubject();
System.out.println("Subject: " + subject);

Object returnObject = Subject.doAs(subject, new PrivilegedExceptionAction<Object>() {
    @Override
    public Object run() throws Exception {
        // Access Kerberos secured services
    }
});
```

# Authenticate using the UserGroupInformation class

[UserGroupInformation](http://hadoop.apache.org/docs/r2.8.3/api/org/apache/hadoop/security/UserGroupInformation.html) class, packaged with *hadoop-common* library, wraps around a JAAS Subject and provides all the required methods to manage Kerberos-based authentications.

This class is of special interest, since it allows the developers not to worry about the JAAS configuration and hides the underlying implementation complexity using the aforementioned LoginContext, Subject, Principal, etc. classes.

A developer can completely handle all the Kerberos authentication options using just the UserGroupInformation class to access secured services, which are covered one by one in the following sections.

## Local ticket cache

To test the authentication using an existing TGT you need to first request and store it in a custom cache location, using the `kinit` command below:

```java
[andriymz@fedora27 ~]$ kinit user@DOMAIN -kt andriy.keytab -c /tmp/cache
[andriymz@fedora27 ~]$ klist -c /tmp/cache
Ticket cache: FILE:/tmp/cache
Default principal: user@DOMAIN

Valid starting       Expires              Service principal
24-03-2019 18:04:30  25-03-2019 18:04:30  krbtgt/DOMAIN@DOMAIN
        renew until 31-03-2019 19:04:30 
```

Then, in your application you need to create an instance of UserGroupInformation class by calling its `getUGIFromTicketCache` method, which expects the TGT cache location and its Kerberos principal as parameters.

```java
UserGroupInformation ugi = UserGroupInformation.getUGIFromTicketCache("/tmp/cache", "user@DOMAIN");
```
UserGroupInformation instance contains the authenticated Subject and User objects, a boolean field indicating if the instance was created using a keytab and another boolean set to true if the instance contains any Kerberos ticket.

{% include figure image_path="/assets/images/kerberos/ugi_1.png" %}

The User class, which implements the previously introduced Principal interface, contains basic information about the logged-in entity, such as Kerberos principal and the corresponding authentication method. 

{% include figure image_path="/assets/images/kerberos/ugi_2.png" %}

Finally, the Subject class contains the logged-in Principals, and a set of public and private credentials, just as mentioned previously. 

Note that the private credentials set has already one entry of KerberosTicket object, which corresponds to the TGT taken from the cache. Before accessing any secured service, the client first needs to request the service ticket from the TGS. These tickets are also populated within the private credentials set of a Subject.
{% include figure image_path="/assets/images/kerberos/ugi_3.png" %}


The UserGroupInformation instance is then used to access the secured Hadoop services. You can now access the secured Hadoop FS by executing the following code:

```java
Configuration conf = new Configuration();
conf.set("fs.defaultFS", "hdfs://host:8020");
conf.set("hadoop.security.authentication", "kerberos");

UserGroupInformation.setConfiguration(conf);

FileStatus[] fsStatus = ugi.doAs(new PrivilegedExceptionAction<FileStatus[]>() {
    @Override
    public FileStatus[] run() throws Exception {
        FileSystem fs = FileSystem.get(conf);
        return fs.listStatus(new Path("/user/andriy"));
    }
});

for(FileStatus fStat: fsStatus){
    System.out.println(fStat.getPath().toString());
}
```

## Keytab 

The previous way of authenticating is fine for short-living applications where a client previously acquired a TGT, however since the TGT is only valid for 24h (by default) and can be renewed up to 7 days (by default) after it was issued, an alternative keytab variant of authentication should be used to ensure that the TGT is renewed accordingly and the application can continue to access secured Hadoop services.

There are two methods allowing to login using a keytab: `loginUserFromKeytab` and `loginUserFromKeytabAndReturnUGI`. The mais difference between them is that the former method returns nothing and sets the currently logged-in user at the UserGroupInformation level, which can be then accessed using the `getCurrentUser` method, and the latter just logs-in without affecting the currently logged-in user.

The example below shows how both methods can be used. Note that this allows us to have multiple logged-in users within the same JVM.

```java
UserGroupInformation.loginUserFromKeytab("bob@DOMAIN", "/home/bob/bob.keytab");
UserGroupInformation ugi = UserGroupInformation.getCurrentUser();

ugi = doAs(subject, new PrivilegedExceptionAction<Object>() {
    @Override
    public Object run() throws Exception {
        // Execute something with Bob as user
    }
});

UserGroupInformation tomUgi = UserGroupInformation.loginUserFromKeytabAndReturnUGI("tom@DOMAIN", "/home/tom/tom.keytab");
tomUgi = doAs(subject, new PrivilegedExceptionAction<Object>() {
    @Override
    public Object run() throws Exception {
        // Execute something with Tom as user
    }
});
```

## TGT renewal

Long-running applications should have its TGT renewed, for this you should call the `checkTGTAndReloginFromKeytab` method. It is up to you to decide when this method should be called, from time to time in a background method or before each privileged action (check this [response](https://stackoverflow.com/questions/34616676/should-i-call-ugi-checktgtandreloginfromkeytab-before-every-action-on-hadoop) from a Hadoop commiter). The ticket is only renewed if its validity reaches 80%, and is a no-op otherwise as we can check in its implementation:

```java
  /**
   * Re-login a user from keytab if TGT is expired or is close to expiry.
   * 
   * @throws IOException
   * @throws KerberosAuthException if it's a kerberos login exception.
   */
  public synchronized void checkTGTAndReloginFromKeytab() throws IOException {
    if (!isSecurityEnabled()
        || user.getAuthenticationMethod() != AuthenticationMethod.KERBEROS
        || !isKeytab) {
      return;
    }
    KerberosTicket tgt = getTGT();
    if (tgt != null && !shouldRenewImmediatelyForTests &&
        Time.now() < getRefreshTime(tgt)) {
      return;
    }
    reloginFromKeytab();
  }
```

## Impersonation

Another important feature of authentication is impersonation using the Hadoop [proxy users](https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/Superusers.html), which enable applications to access resources on the cluster on behalf of another user. 

Only the proxy user needs to have Kerberos credentials and any user can be impersonated. 


```java
//Create ugi for joe. The login user is 'super'.
UserGroupInformation ugi = UserGroupInformation.createProxyUser("joe", UserGroupInformation.getLoginUser());

ugi.doAs(new PrivilegedExceptionAction<Object>() {
  public Object run() throws Exception {
    // Execute an action on behalf of Joe
  }
}
```

The control of which users are capable of impersonating others is done by specifying as below in the Hadoop `core-site.xml` configuration file. Since it is a powerful setting, only trusted services should be added to the proxy user setup. A wildcard value * may be used to allow impersonation from any host or of any user.

```xml
<property>
    <name>hadoop.proxyuser.super.hosts</name>
    <value>host1,host2</value>
</property>
<property>
    <name>hadoop.proxyuser.super.groups</name>
    <value>group1,group2</value>
</property>
```

## Delegation Tokens

The last topic I would like to cover are Hadoop Delegation Tokens. Delegation Tokens were introduced as an authentication method to complement Kerberos authentication, instead of solely relying on TGTs and service tickets. To fully understand how delegation tokens work you can check this great [blog post](https://blog.cloudera.com/blog/2017/12/hadoop-delegation-tokens-explained/), but the main point is that a client (distributed job submitter) can request a Delegation Token from a secured service (e.g. HDFS) and pass it to the workers, which can authenticate as, and run jobs on behalf of, the client.
 
Delegation Tokens also have an expiration time and require periodic renewals to keep their validity, however if compromised it would only grant access to a single service until the token expires or is cancelled.
Delegation Tokens eliminate the need to distribute a keytab over the network, which, if compromised, would grant access to all services and logically cause more damage.


### Request a HDFS Delegation Token

HDFS Delegation Token can be obtained by calling the [FileSystem](https://hadoop.apache.org/docs/r2.8.2/api/org/apache/hadoop/fs/FileSystem.html)'s *addDelegationTokens* method. This method receives the user allowed to renew the delegation tokens and the Credentials object in which the obtained tokens are added. The Delegation Tokens can only be obtained within Kerberos authenticated context, *i.e.* when UserGroupInformation's authentication method is *KERBEROS*.

```java
Configuration conf = new Configuration();
conf.set("fs.defaultFS","hdfs://host:8020");
conf.set("hadoop.security.authentication", "kerberos");

UserGroupInformation.setConfiguration(conf);

UserGroupInformation ugi = UserGroupInformation.loginUserFromKeytabAndReturnUGI("user@DOMAIN", "/tmp/andriy.keytab");

// Credentials object to be populated with Delegation Tokens
Credentials credentials = new Credentials();

ugi.doAs(new PrivilegedExceptionAction<Void>() {
    @Override
    public Void run() throws Exception {
        FileSystem fs = FileSystem.get(conf);
        fs.addDelegationTokens("super", credentials);
        return null;
    }
});
```

Credentials contain a collection of tokens implementing the [Token](https://hadoop.apache.org/docs/r2.9.1/api/org/apache/hadoop/yarn/api/records/Token.html) interface, containing the elements below:

{% include figure image_path="/assets/images/kerberos/token_1.png" %}

It is also possible to get the underlying class implementing the Token interface by calling its *decodeIdentifier* method. In this example the HDFS Delegation Token is represented by the [DelegationTokenIdentifier](https://hadoop.apache.org/docs/current/hadoop-mapreduce-client/hadoop-mapreduce-client-core/apidocs/org/apache/hadoop/mapreduce/security/token/delegation/DelegationTokenIdentifier.html) class containing all the token details:

{% include figure image_path="/assets/images/kerberos/token_2.png" %}

### Serializing Delegation Tokens

Delegation Tokens objects are stored in a [Credential](https://hadoop.apache.org/docs/current/api/org/apache/hadoop/security/Credentials.html) class, which not only provides methods to add, get and remove tokens, but also some convenience methods for storing and reading the Credentials state through [DataOutputStream](https://docs.oracle.com/javase/7/docs/api/java/io/DataOutputStream.html) and [DataInputStream](https://docs.oracle.com/javase/7/docs/api/java/io/DataInputStream.html).

Serializing a Credential object to `byte[]` can be achived by the code below: 

```java
// Create data output stream
ByteArrayOutputStream baos = new ByteArrayOutputStream();
DataOutputStream dos = new DataOutputStream(baos);

// Serialize to byte[]
credentials.writeTokenStorageToStream(dos);
byte[] credentialsBytes = baos.toByteArray();

// Create data input stream
ByteArrayInputStream bais = new ByteArrayInputStream(credentialsBytes);
DataInputStream dis = new DataInputStream(bais);

// Deserialize from byte[]
Credentials recoveredCredentials = new Credentials();
recoveredCredentials.readTokenStorageStream(dis);
```

You can also store the Credentials to HDFS by running the following code, of course in a secure Kerberos context:

```java
Configuration conf = new Configuration();
conf.set("fs.defaultFS","hdfs://host:8020");
conf.set("hadoop.security.authentication", "kerberos");

...

Path credentialsCachePath = new Path("/user/andriy/credentials_cache");

credentials.writeTokenStorageFile(credentialsCachePath, conf);

Credentials recoveredCredentials2 = Credentials.readTokenStorageFile(credentialsCachePath, conf);
```

Spark actually handles the credential delegation this way. Application Master periodically requests Delegation Tokens from HDFS, Hive and HBase services, and stores the Credential object containing them to HDFS. Then each Executor reads the fresh Credentials from HDFS and updates its UserGroupInformation. This allows Spark to work against a secured cluster where only the Client and the Application Master have Kerberos credentials. All those details are explained in [How Spark Uses Kerberos Authentication](/kerberos/how-spark-uses-kerberos-authentication) post.


### Authenticate using Delegation Tokens

Once you have a Credentials object you can update the UserGroupInformation with the Delegation Tokens contained in it. This can be done with *addCredentials* or *addToken* methods, depending if you want to add all the tokens at once or only specific ones.

By executing the code below you will be able to access the HDFS service because HDFS Delegation Token is added to the created UserGroupInformation, not requiring any login from keytab or ticket cache.  

Notice that the UserGroupInformation instance was created with a *createRemoteUser* method, which creates a user from a login name. It is intended to be used for remote users in RPC, since it won't have any credentials.

```java
UserGroupInformation newUgi = UserGroupInformation.createRemoteUser("user");

// Option 1: Add all the tokens contained in Credentials object
newUgi.addCredentials(credentials);

// Option 2: Add the tokens one by one
Collection<Token<? extends TokenIdentifier>> tokens = credentials.getAllTokens();

for(Token<?> token : tokens) {
    if(token.getKind().toString().equals("HDFS_DELEGATION_TOKEN")){
        newUgi.addToken(token);
    }
}

FileStatus[] fsStatus = newUgi.doAs(new PrivilegedExceptionAction<FileStatus[]>() {
    @Override
    public FileStatus[] run() throws Exception {
        FileSystem fs = FileSystem.get(conf);
        return fs.listStatus(new Path("/user/andriy"));
    }
});

for(FileStatus fStat: fsStatus){
    System.out.println(fStat.getPath().toString());
} 
```

In HDFS Audit logs you can confirm that the originated *listStatus* request was performed under *user@DOMAIN* principal and *TOKEN* authentication method.

```text
INFO FSNamesystem.audit: allowed=true   ugi=user@DOMAIN (auth:TOKEN)        ip=/148.63.81.239       cmd=listStatus  src=/user/andriy dst=null        perm=null       proto=rpc
```
