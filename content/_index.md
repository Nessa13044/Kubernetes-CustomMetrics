---
title : "Session Management"
date :  "`r Sys.Date()`" 
weight : 1 
chapter : false
---
# SCALING APPLICATION WITH KUBERNETES

### Overall
 - Horizontal Pod Autoscaler (HPA) in Kubernetes can use Metric Server to scale pods automatically, only supporting metrics such as CPU, memory. Therefore, if applications want to automatically scale according to their own metrics, Kubernetes will not support it. 
 - Problem: I have applications running on tomcat installed on kubernetes. I want these applications to automatically scale to handle the rapid increase in requests. 

{{% notice info %}}
SOLUTION: I use the metric is **tomcat_requestcount_total** and Prometheus tools, Prometheus-Adapter combined with HPA to scale according to Custom metrics.
{{% /notice %}}

#### This is the overall architecture of scale.
![adapter](/images/3.connect/adapter.png)

### Content
 1. [Kubernetes Architecture](1-KubernetesArchitecture/)
 2. [Setup Lab](2-SetupLab/)
 3. [Monitoring](3-accessibilitytoinstances/)
 4. [Manage session logs](4-cleanup/)
 
