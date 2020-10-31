---
title:  "SPNego Authentication Fails to HTTPS Service"
excerpt: "If you are using the HttpClient library of version 4.5.2 to make HTTP requests to a backend server with SSL and SPNego, and the requests are unexpectedly failing, you may be dealing with a known bug."
classes: wide
toc: true
categories: [kerberos]
toc: true
tags: [spnego]
author_profile: true
---

If you are using the [HttpClient](https://hc.apache.org/httpcomponents-client-4.5.x/) library of version 4.5.2 to make HTTP requests to a backend server with SSL and SPNego, and the requests are unexpectedly failing with the errors below, you may be dealing with a known bug. 

Application log:

```
GSSException: No valid credentials provided (Mechanism level: Fail to create credential. (63) - No service creds)
```

KDC log: 

```
TGS_REQ (4 etypes {18 17 16 23}) 10.159.4.3: LOOKING_UP_SERVER: authtime 0,  someinstance@REALM for HTTP
S/hostname@REALM, Server not found in Kerberos database
```

Before blaming the library, make sure your backend server is properly configured and the SPNego authentication does work when requests are made using cURL, as follows: 

```bash
# Do a kinit to have a TGT is your local ticket cache
kinit

# Make a request 
curl -i -k --negotiate -u : -v -H "Content-Type: application/json" -X POST -d '{}' https://host.example.com:443/api/test
```

## Background

When the client makes a request to a backend server with SPNego authentication, the following steps are involved during the Negotiation:

1. Client sends an HTTP request to the server;
2. SPNego authentication in the server answers the client browser with an HTTP 401 challenge header that contains the `Authenticate: Negotiate` status;
3. The client recognizes the negotiate header and parses the requested URL for the host name. The client uses the **host name** to form the **target Kerberos Principal** `HTTP/host.example.com` to request a Kerberos service ticket from the Kerberos ticket-granting service (TGS) in the Kerberos KDC (TGS_REQ). In response the **TGS issues a Kerberos service ticket** (TGS_REP) to the client;
4. The client browser responds to `Authenticate: Negotiate` challenge with the SPNego token, containing the service ticket obtained in the previous step, in the request HTTP header `Authorization: Negotiate [token]`, where `[token]` represents the SPNego token encoded in Base64;
5. Server reads the HTTP header with the SPNego token, validates the token, and gets the identity (principal) of the user;
6. After validating the identity and performing the authorization checks, the server sends the response with an HTTP 200. This response also includes a session cookie (JSESSIONID) to be used for subsequent requests that are part of this authenticated session.


## Problem

The problem with the version 4.5.2 of the HttpClient library is that in **step 3** it uses `HTTPS/host.example.com` as the target Kerberos Principal when the backend server has SSL. This behavior is not conformant with the [standards](https://sites.google.com/a/chromium.org/dev/developers/design-documents/http-authentication) and was incorrectly reported as issue in [[HTTPCLIENT-1712] SPNego Authentication to HTTPS service](https://issues.apache.org/jira/browse/HTTPCLIENT-1712).

Unfortunately, version 4.5.2 patched this issue and the projects depending on it experience the `Server not found in Kerberos database` error, because `HTTP/host.example.com` and `HTTPS/host.example.com` are two different Kerberos Principals, and only the former should be present in the KDC.

For example, Apache Knox also reported an issue related with the dependency on this version of HttpClient, see [[KNOX-762] Remove dependency on httpcomponents httpclient 4.5.2](https://issues.apache.org/jira/browse/KNOX-762).


The changes that introduced this behavior were in the [GGSSchemeBase](https://github.com/apache/httpcomponents-client/blob/4.5.2/httpclient/src/main/java/org/apache/http/impl/auth/GGSSchemeBase.java) class, see [commit](https://github.com/apache/httpcomponents-client/commit/1d50c1a1a16ee6f9c5930d966f1b234d98bf261f#diff-e60a51dc1af8a64accbc2e0423827c22):

{% include figure image_path="/assets/images/kerberos/spnego/change1.png" %}

In this version the service is `HTTPS` if the SSL is enabled, and `HTTP` otherwise. 

This change was then rolled back in version 4.5.3, see [commit](https://github.com/apache/httpcomponents-client/commit/1b41b52b1481067dab5672bc5c1161adc4558b06#diff-e60a51dc1af8a64accbc2e0423827c22): 

{% include figure image_path="/assets/images/kerberos/spnego/change2.png" %}


## Solution

The easier solution, if possible, is just not using the version 4.5.2 of HttpClient. Upgrading to version 4.5.3 solves the problem.

If upgrading the library is not possible, you can also do a tricky workaround by creating two additional classes in your project to override the buggy method: [FixedSPNegoScheme.java](https://github.com/johrstrom/cloud-meter/blob/master/cloud-meter-protocols/src/main/java/org/apache/jmeter/protocol/http/control/FixedSPNegoScheme.java) and [FixedSPNegoSchemeFactory.java](https://github.com/johrstrom/cloud-meter/blob/dbfa4b667b211275f82f057687568df69a8faa12/cloud-meter-protocols/src/main/java/org/apache/jmeter/protocol/http/control/FixedSPNegoSchemeFactory.java). Then use the `FixedSPNegoSchemeFactory` class instead of the original `SPNegoSchemeFactory` in your implementation.


## References

[IBM Knowledge Center - SPNego](https://www.ibm.com/support/knowledgecenter/en/SSEQTP_liberty/com.ibm.websphere.wlp.doc/ae/cwlp_spnego.html)

[Kerberos Wireshark Captures: A SPNego Example](https://medium.com/@robert.broeckelmann/kerberos-wireshark-captures-a-spnego-example-e22e6b1d662a)

[The MIT Kerberos Administrator’s How-to Guide ](https://www.kerberos.org/software/adminkerberos.pdf)
