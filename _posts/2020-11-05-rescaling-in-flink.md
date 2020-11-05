---
title:  "Rescaling in Flink"
excerpt: "How to rescale a running Flink job? Rescaling is useful to better use computational resources when your application does not have the same workload at all times."
classes: wide
categories: [flink]
toc: true
---

Rescaling a running Flink job is useful to better use computational resources when your application does not have the same workload at all times. In theory it means that at some point you should be able to scale up or down, by adding or removing _TaskManagers_ (worker processes) on a Flink cluster.

Currently there is no way of "automagically" rescaling a running Flink job, i.e changing it's _parallelism_ when there are changes on available _TaskManagers_ to execute the subtasks. Even if you spawn more _TaskManagers_ and they get registered at the _JobManager_, the already running job will still use the same level of parallelism. 

This post covers some findings about rescaling. The latest version of Flink is 1.11.2 at the time of this writing.

## Removal of Job Rescaling from CLI and REST API

According to [this malining list](http://apache-flink-user-mailing-list-archive.2336050.n4.nabble.com/DISCUSS-Temporarily-remove-support-for-job-rescaling-via-CLI-action-quot-modify-quot-td27447.html), the experimental feature of modifying the parallelism of an already running Flink job was removed. 

If you compare CLI (Command Line Interface) documentations of versions [1.8](https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/cli.html) and [1.9](https://ci.apache.org/projects/flink/flink-docs-release-1.9/ops/cli.html) you can see that the command below was removed ([FLINK-12312](https://issues.apache.org/jira/browse/FLINK-12312)) from the CLI:
```
flink modify <jobID> -p <newParallelism>
```

In older versions this command would rescale an already running job with the specified by the user new _parallelism_. This is also supported by Flink's REST API and it's interesting that the two endpoints for it are still available in latest 1.11.2 version, having the specification below:

{% include figure image_path="/assets/images/flink/rescaling/rescale_rest.png" %}

But the endpoint `/jobs/:jobId/rescaling` is not working as well and returns an error:

```json
{
  "errors": [
    "org.apache.flink.runtime.rest.handler.RestHandlerException: Rescaling is temporarily disabled. See FLINK-12312 (..)"
  ]
}
```

## Reactive Container Mode

Apparently there is an active development ([FLINK-10407](https://issues.apache.org/jira/browse/FLINK-10407)) on a feature called _Reactive Container Mode_ in which according to the description makes a Flink cluster "react to newly available resources (e.g. started by an external service) and make use of them by rescaling the existing job."

You can check all the details of this feature in their [design document](https://docs.google.com/document/d/1XKDXnrp8w45k2jIJNHxpNP2Zmr6FGru3H0dTHwYWJyE/edit) and possibly it will be released in version 1.12, as pointed in [release document](https://cwiki.apache.org/confluence/display/FLINK/1.12+Release) as "Reactive-scaling mode" feature.

## Rescaling with a Job Restart
   
Currently the only way of rescaling a Flink job is by doing a graceful shutdown with a _savepoint_ and restarting the job from the saved _savepoint_ with the new _parallelism_.

Netflix implemented their own autoscaling algorithm by following this approach and deciding how to scale based on analysis from collected metrics. Check their
[Autoscaling Flink at Netflux](https://www.youtube.com/watch?v=NV0jvA5ZDNc) talk by Timothy Farkas.

More on Flink's _Reactive Container Mode_ in [Future of Apache Flink Deployments: Containers, Kubernetes and More - Till Rohrmann](https://www.youtube.com/watch?v=WeHuTRwicSw) talk by Till Rohrmann.

## Resources

[Rescaling endpoints on Flink's REST API](https://ci.apache.org/projects/flink/flink-docs-release-1.11/monitoring/rest_api.html#jobs-jobid-rescaling)

[[FLINK-12312] Temporarily disable CLI command for rescaling](https://issues.apache.org/jira/browse/FLINK-12312)

[[FLINK-10407] Reactive Container Mode](https://issues.apache.org/jira/browse/FLINK-10407)

[Flink Design Document: Reactive Container Mode](https://docs.google.com/document/d/1XKDXnrp8w45k2jIJNHxpNP2Zmr6FGru3H0dTHwYWJyE/edit#heading=h.wvy3b8sj26gj)

[Deploy Flink Job Cluster on Kubernetes](http://shzhangji.com/blog/2019/08/24/deploy-flink-job-cluster-on-kubernetes)
