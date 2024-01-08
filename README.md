# resource-optimization-guidelines

A short set of instructions and queries to find out the limits/requests that better work for your cluster

- [resource-optimization-guidelines](#resource-optimization-guidelines)
	- [General recommendations](#general-recommendations)
	- [Next steps](#next-steps)
		- [1 - Get more details on what limits and requests are and what they’re used for](#1---get-more-details-on-what-limits-and-requests-are-and-what-theyre-used-for)
			- [Sources](#sources)
		- [2 - Get a broader view to find the biggest offenders](#2---get-a-broader-view-to-find-the-biggest-offenders)
			- [How to find the biggest offenders](#how-to-find-the-biggest-offenders)
				- [Namespaces](#namespaces)
				- [Pods](#pods)
		- [3 - Find the biggest offender and start with that one](#3---find-the-biggest-offender-and-start-with-that-one)
			- [3.1 - What data do you need to have at hand before moving on?](#31---what-data-do-you-need-to-have-at-hand-before-moving-on)
			- [3.2 - Based off of that data, try to aim to a more reasonable request/limit](#32---based-off-of-that-data-try-to-aim-to-a-more-reasonable-requestlimit)
			- [3.3 - Let it run for a few days and observe the behaviour](#33---let-it-run-for-a-few-days-and-observe-the-behaviour)
		- [4 - Repeat](#4---repeat)
		- [5 - Adjust](#5---adjust)

## General recommendations

1. One way to improve resource allocation is to target the biggest offenders first. By analysing cluster metrics, it is possible to identify specific nodes in the cluster that could be saved with some changes.
   1. For optimal results, reviewing the historical Prometheus data and identifying values that align more closely with the current usage patterns is advisable.
   2. If this approach proves successful, a plan could be implemented to make it easier for other teams to monitor and track their usage over time
   3. Once this analysis has been completed, be aware of CPU and memory reaching their respective limits, as kubernetes/openshift will throttle CPU performance if the pod reaches 100% and memory usage over limit will result in OOMK pods.
2. Consider switching from on-demand to contracted pricing, as this change could provide the most substantial benefits for all clusters.

## Next steps

### 1 - Get more details on what limits and requests are and what they’re used for

The most prevalent issue lies within each deployment and pod definition, specifically the requests and limits part. So before we begin we need to read the following:

- [ ] The official kubernetes docs on requests and limits [here](https://docs.openshift.com/container-platform/4.13/nodes/clusters/nodes-cluster-resource-configure.html)
- [ ] Configuring cluster memory to meet container memory and risk requirements [here](https://docs.openshift.com/container-platform/4.13/nodes/clusters/nodes-cluster-limit-ranges.html)
- [ ] The official openshift docs on limit ranges [here](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

The very first step before taking any action is to analyse what we have as of today. To do so we can make use of a few sources at our disposal:

#### Sources

1. Custom metrics and logs: Do you have a custom grafana server running? A loki instance or custom prometheus? Maybe an elastic stack? Great! Some of this information might be incredibly useful when deciding which action to take.
2. The [embedded openshift prometheus instance](https://docs.openshift.com/container-platform/4.13/monitoring/querying-metrics.html): Openshift comes with a built-in prometheus instance, which you can use and query right away using PromQL from your openshift web console! Head over to “Observe” and have a look at the default queries and graphs there!
3. Your own dev teams: This one is absolutely crucial for custom software deployed on your openshift cluster. No one else should know the exact memory and CPU (and network, disk…) requirements better than their developers. This exercise should be done by dev teams when possible, so each team is in charge of adjusting limits and requests whenever a new feature is added or deprecated.

### 2 - Get a broader view to find the biggest offenders

We need to find the biggest offenders, a.k.a the Deployments that are wasting the most CPU/memory requests by not using them. Once we fix those, we can slowly move on to less overprovisioned Deployments.

#### How to find the biggest offenders

Let’s start simple, we can query the embedded prometheus instance by heading over to your Red Hat Openshift Container Platform web console > Observe > Metrics and pasting the following:

##### Namespaces

> *Top 3 namespaces by CPU request %*

```promql
bottomk(3,sum(node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate{cluster="",namespace!~".*openshift.*"}) by (namespace) / sum(namespace_cpu:kube_pod_container_resource_requests:sum{cluster="",namespace!~".*openshift.*"}) by (namespace))
```

> *Top 3 namespaces using the most CPU limit %*

```promql
topk(3,sum(node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate{cluster="",namespace!~".*openshift.*"}) by (namespace) / sum(namespace_cpu:kube_pod_container_resource_limits:sum{cluster="",namespace!~".*openshift.*"}) by (namespace))
```

> *Top 3 namespaces using the least Memory request %*

```promql
bottomk(3,sum(container_memory_rss{cluster="", container!="",namespace!~".*openshift.*"}) by (namespace) / sum(namespace_memory:kube_pod_container_resource_requests:sum{cluster="",namespace!~".*openshift.*"}) by (namespace))
```

> *Top 3 namespaces using the most memory limit %*

```promql
topk(3,sum(container_memory_rss{cluster="", container!="",namespace!~".*openshift.*"}) by (namespace) / sum(namespace_memory:kube_pod_container_resource_limits:sum{cluster="",namespace!~".*openshift.*"}) by (namespace))
```

##### Pods

> *Top 3 pods using the least CPU request %*

```promql
bottomk(3,(sum(node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate{cluster="",namespace!~".*openshift.*", container!="", image!=""}) by (container) / sum(cluster:namespace:pod_cpu:active:kube_pod_container_resource_requests{cluster="",namespace!~".*openshift.*"}) by (container)))
```

> *Top 3 pods using the most CPU limit %*

```promql
topk(3,(sum(node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate{cluster="",namespace!~".*openshift.*", container!="", image!=""}) by (container) / sum(cluster:namespace:pod_cpu:active:kube_pod_container_resource_limits{cluster="",namespace!~".*openshift.*"}) by (container)))
```

> *Top 3 pods using the least memory request %*

```promql
bottomk(3,(sum(container_memory_working_set_bytes{cluster="",namespace!~".*openshift.*", container!="", image!=""}) by (container) / sum(cluster:namespace:pod_memory:active:kube_pod_container_resource_requests{cluster="",namespace!~".*openshift.*"}) by (container)))
```

> *Top 3 pods using the most memory limit %*

```promql
topk(3,(sum(container_memory_working_set_bytes{cluster="",namespace!~".*openshift.*", container!="", image!=""}) by (container) / sum(cluster:namespace:pod_memory:active:kube_pod_container_resource_limits{cluster="",namespace!~".*openshift.*"}) by (container)))
```

With all of this information for the biggest offender/s we can move on to the next section.

### 3 - Find the biggest offender and start with that one

#### 3.1 - What data do you need to have at hand before moving on?

- [ ] Past usage
  - [ ] The 5 day average of actual Memory usage for the deployment
  - [ ] The 5 day average of actual CPU usage for the deployment
  - [ ] The highest peak of actual memory usage during a 5 day period (a.k.a. Memory spikes)
  - [ ] The highest peak of actual CPU usage during a 5 day period (a.k.a CPU spikes)
- [ ] Expected usage (done by dev teams)
  - [ ] Expected use of memory
  - [ ] Expected use of CPU
  - [ ] Expected maximum use of memory and under what conditions
  - [ ] Expected maximum use of CPU and under what conditions

#### 3.2 - Based off of that data, try to aim to a more reasonable request/limit

Work with your dev teams to aim for a more appropriate CPU and memory request. Remember that memory request means that the scheduler will save up that amount of the specific resource to only that pod. A pod with a very low % usage of that resource will result in unused memory or CPU for the entire cluster.
Once requests are done, make sure to set an appropriate limit (this means that the scheduler might kill the pod if it goes over the memory limit, or throttle CPU performance if it goes over the CPU limit). Leave enough room for the pod to breathe, but make sure that there is a limit set so in case of a memory leak or misbehaving application, the pod does not take up the entire node.

#### 3.3 - Let it run for a few days and observe the behaviour

### 4 - Repeat

Repeat with all namespaces and deployments (with the corresponding dev team)

### 5 - Adjust

After a sensible amount of time has passed, review usage data and adjust requests/limits accordingly.
