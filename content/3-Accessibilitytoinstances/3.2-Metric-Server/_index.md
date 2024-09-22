---
title : "Prometheus - Adapter"
date : "`r Sys.Date()`"
weight : 2
chapter : false
pre : " <b> 3.2. </b> "
---


- Although HPA is installed by default in K8s, to get information for monitoring (CPU, Memory), but if you want to extend by other parameters like jmx then metricserver will not support. So i use prometheus-adapter to scale by metric "tomcat_requestcount_total"

#### Architecture
![adapter](/images/3.connect/adapter.png)

#### 1.Download Prometheus Adapter Manifest
- Use below command to clone yaml file.
```
git clone https://github.com/kubernetes-sigs/prometheus-adapter.git
```
- Move to folder deploy/manifests and you'll see all the necessary .yaml file
- I need to edit some files to fit this lab. First, open the file **api-service.yaml**
```
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  labels:
    app.kubernetes.io/component: metrics-adapter
    app.kubernetes.io/name: prometheus-adapter
    app.kubernetes.io/version: 0.12.0
  #name: v1beta1.metrics.k8s.io
  name: v1beta1.custom.metrics.k8s.io
spec:
  #group: metrics.k8s.io
  group: custom.metrics.k8s.io
  groupPriorityMinimum: 100
  insecureSkipTLSVerify: true
  service:
    name: prometheus-adapter
    namespace: monitoring
  version: v1beta1
  versionPriority: 100
```
- since using custom metrics i changed kubernetes support API to **custom.metrics.k8s.io**

- Next, replace the configmap.yaml file below to determines which metrics to expose and specifies each of the steps the adapter needs to take to expose a metric in the API.

```
apiVersion: v1
data:
  config.yaml: |
    rules:
    #- seriesQuery: 'tomcat_requestcount_total'
    - seriesQuery: '{__name__=~"^tomcat_.*",instance!=""}'
      #resources: {overrides: {job: {resource: "job"}}}
      resources:
        overrides:
          instance: {resource: "node"}
          job: {resource: "job"}
          kubernetes_namespace: {resource: "namespace"}
          kubernetes_pod_name: {resource: "pod"}
      name:
        matches: "tomcat_requestcount_total"
        #matches: "^tomcat_(.*)$"
      metricsQuery: "sum(rate(<<.Series>>{<<.LabelMatchers>>}[3m])*180) by (<<.GroupBy>>)"
    externalRules:
    - seriesQuery: 'tomcat_requestcount_total'
      resources:
        overrides:
          instance: {resource: "node"}
          job: {resource: "job"}
          kubernetes_namespace: {resource: "namespace"}
          kubernetes_pod_name: {resource: "pod"}
      name:
        matches: "tomcat_requestcount_total"
        as: "tomcat_requestcount_total"
      metricsQuery: "sum(rate(<<.Series>>{<<.LabelMatchers>>}[3m])*180) by (<<.GroupBy>>)"

kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/component: metrics-adapter
    app.kubernetes.io/name: prometheus-adapter
    app.kubernetes.io/version: 0.12.0
  name: adapter-config
  namespace: monitoring
```
- To explain a bit, Each rule can be divided into four parts:
+ **Discovery** which specifies how the adapter should find all Prometheus metrics for this rule.

+ **Association** which specifies how the adapter should determine which Kubernetes resources a particular metric is associated with.

+ **Naming** which specifies how the adapter should expose the metric in the custom metrics API.

+ **Querying** which specifies how a request for a particular metric on one or more Kubernetes objects should be turned into a query to Prometheus.

- [Read more on](https://github.com/kubernetes-sigs/prometheus-adapter/blob/master/docs/config.md)


#### 2. Create Prometheus Adapter

- Follow command to complete deploy adapter.
```
kubectl create -f prometheus-adapter/deploy/manifests/
```
- As we can see, prometheus-adapter to deploy success and running on cluster 
```
~# kubectl get pod -A -o wide 
NAMESPACE     NAME                                    READY   STATUS    RESTARTS   AGE   IP              NODE         NOMINATED NODE   READINESS GATES
app           tomcat-deployment-7fb69d4749-fnpd4      1/1     Running   0          36m   10.0.1.99       k8s-worker   <none>           <none>
kube-system   cilium-8phqv                            1/1     Running   0          76m   172.22.6.150    k8s-worker   <none>           <none>
kube-system   cilium-bgvjv                            1/1     Running   0          76m   172.22.11.124   k8s-master   <none>           <none>
kube-system   cilium-operator-79bbbc6f56-sdlwq        1/1     Running   0          76m   172.22.6.150    k8s-worker   <none>           <none>
kube-system   coredns-6f6b679f8f-6ljtz                1/1     Running   0          78m   10.0.0.8        k8s-master   <none>           <none>
kube-system   coredns-6f6b679f8f-btcxx                1/1     Running   0          78m   10.0.0.114      k8s-master   <none>           <none>
kube-system   etcd-k8s-master                         1/1     Running   0          79m   172.22.11.124   k8s-master   <none>           <none>
kube-system   kube-apiserver-k8s-master               1/1     Running   0          79m   172.22.11.124   k8s-master   <none>           <none>
kube-system   kube-controller-manager-k8s-master      1/1     Running   30         79m   172.22.11.124   k8s-master   <none>           <none>
kube-system   kube-scheduler-k8s-master               1/1     Running   30         79m   172.22.11.124   k8s-master   <none>           <none>
monitoring    prometheus-adapter-777cb6d9d8-5lpp8     1/1     Running   0          32m   10.0.1.58       k8s-worker   <none>           <none>
monitoring    prometheus-deployment-5b9f6c9dd-z8s9f   1/1     Running   0          36m   10.0.1.30       k8s-worker   <none>           <none>
```
- Let’s check what all custom metrics are available.
```
~# kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/" | jq .
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "custom.metrics.k8s.io/v1beta1",
  "resources": [
    {
      "name": "namespaces/tomcat_requestcount_total",
      "singularName": "",
      "namespaced": false,
      "kind": "MetricValueList",
      "verbs": [
        "get"
      ]
    },
    {
      "name": "pods/tomcat_requestcount_total",
      "singularName": "",
      "namespaced": true,
      "kind": "MetricValueList",
      "verbs": [
        "get"
      ]
    },
    {
      "name": "nodes/tomcat_requestcount_total",
      "singularName": "",
      "namespaced": false,
      "kind": "MetricValueList",
      "verbs": [
        "get"
      ]
    },
    {
      "name": "jobs.batch/tomcat_requestcount_total",
      "singularName": "",
      "namespaced": true,
      "kind": "MetricValueList",
      "verbs": [
        "get"
      ]
    }
  ]
}
```

- We can see that **tomcat_requestcount_total** metric is available on all deployment tomcat. Now, let’s check the current value of this metric.

```
~# kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/app/pods/*/tomcat_requestcount_total" | jq .
{
  "kind": "MetricValueList",
  "apiVersion": "custom.metrics.k8s.io/v1beta1",
  "metadata": {},
  "items": [
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "app",
        "name": "tomcat-deployment-7fb69d4749-fnpd4",
        "apiVersion": "/v1"
      },
      "metricName": "tomcat_requestcount_total",
      "timestamp": "2024-09-16T15:53:02Z",
      "value": "0",
      "selector": null
    }
  ]
}
```
- Success, this is what we need to scale in the next part.
