---
title : "Horizontal Pod Autoscaling - HPA"
date : "`r Sys.Date()`"
weight : 3
chapter : false
pre : " <b> 3.3. </b> "
---
- Custom metrics should be labeled and exposed in a way that HPA can query.
- To use custom metrics, create or modify an HPA resource. Below is an example YAML configuration for HPA that scales based on a custom metric:
```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-tomcat
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: tomcat-deployment
  minReplicas: 1
  maxReplicas: 3
  metrics:
  - type: Pods
    pods:
      metric:
        name: "tomcat_requestcount_total"
      target:
        type: AverageValue
        averageValue: 10000m
```
- Create HPA with the following command.
```
kubectl create -f hpa-tomcat.yaml -n app
```
{{% notice info %}}
Note: You must deploy the hpa in the same namespaces as the pods you want to scale.
{{% /notice %}}

- Now, let's check the current status of HPA as follows.
```
~# kubectl get hpa -A 
NAMESPACE   NAME         REFERENCE                      TARGETS   MINPODS   MAXPODS   REPLICAS  
app         hpa-tomcat   Deployment/tomcat-deployment   0/10      1         3         1          
```

```
~# kubectl describe hpa -A
Name:                                   hpa-tomcat
Namespace:                              app
Labels:                                 <none>
Annotations:                            <none>
CreationTimestamp:                      Mon, 16 Sep 2024 16:42:58 +0000
Reference:                              Deployment/tomcat-deployment
Metrics:                                ( current / target )
  "tomcat_requestcount_total" on pods:  14400m / 10
Min replicas:                           1
Max replicas:                           3
Deployment pods:                        1 current / 2 desired
Conditions:
  Type            Status  Reason              Message
  ----            ------  ------              -------
  AbleToScale     True    SucceededRescale    the HPA controller was able to update the target scale to 2
  ScalingActive   True    ValidMetricFound    the HPA was able to successfully calculate a replica count from pods metric tomcat_requestcount_total
  ScalingLimited  False   DesiredWithinRange  the desired count is within the acceptable range
Events:
  Type     Reason                        Age                   From                       Message
  ----     ------                        ----                  ----                       -------
  Normal   SuccessfulRescale             5s (x2 over 4d23h)    horizontal-pod-autoscaler  New size: 2; reason: pods metric tomcat_requestcount_total above target

```

