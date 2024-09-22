---
title : "Prometheus"
date : "`r Sys.Date()`"
weight : 1
chapter : false
pre : " <b> 3.1. </b> "
---

#### Prometheus Monitoring Setup on Kubernetes
1. Prometheus Kubernetes Manifest Files
- I will use these sample configuration files hosted on Github and reconfigure a bit. You can clone the repository using the following command. 
```
git clone https://github.com/techiescamp/kubernetes-prometheus
```

2. Create a Namespace & ClusterRole
- First, we will create a Kubernetes namespace for all our monitoring components. If you don’t create a dedicated namespace, all the Prometheus kubernetes deployment objects get deployed on the default namespace.
- Execute the following command to create a new namespace named monitoring.

```
kubectl create namespace monitoring
```
{{% notice note %}}
Prometheus uses Kubernetes APIs to read all the available metrics from Nodes, Pods, Deployments, etc. For this reason, we need to create an **RBAC** policy with **read access** to required API groups and bind the policy to the **monitoring** namespace.
{{% /notice %}}

- Create RBAC from default **clusterRole.yaml** file
- **clusterRole.yaml**
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: default
  namespace: monitoring
```
- A little explanation about the above role, the default configurations are more than enough for this lab, you can see that get, list and watch permissions are added to nodes, service endpoints, pods and ingress. The role binding is bound to the monitor namespace. If you have any use case to get metrics from any other object, you need to add that to this cluster role.

- Create the role using the following command.
```
kubectl create -f clusterRole.yaml
```


3. Create a Config Map To Externalize Prometheus Configurations
- All configurations for Prometheus are part of **prometheus.yaml** file and all the alert rules for Alertmanager are configured in **prometheus.rules**.
  + **prometheus.yaml**: This is the main Prometheus configuration which holds all the scrape configs, service discovery  details, storage locations, data retention configs, etc.
  + **prometheus.rules**: This file contains all the Prometheus alerting rules

- By externalizing Prometheus configs to a Kubernetes config map, you don’t have to build the Prometheus image whenever you need to add or remove a configuration. You need to update the config map and restart the Prometheus pods to apply the new configuration.

- The config map with all the Prometheus scrape config and alerting rules gets mounted to the Prometheus container in **/etc/prometheus** location as **prometheus.yaml** and **prometheus.rules** files.

- In this lab i don't use **rometheus.rules**. maybe i will set it up in next lab

- Open the config-map.yaml file and add the yaml below. I wrote this to get the tomcat metrics that I want to get.
```
      - job_name: 'tomcat'
        kubernetes_sd_configs:
        - role: endpoints
        relabel_configs:

        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod_name

        - source_labels: [__meta_kubernetes_endpoints_name]
          regex: 'tomcat-service'
          action: keep
        - source_labels: [__meta_kubernetes_endpoint_port_name]
          regex: 'prometheus'
          action: keep
```
- Execute the following command to create the config map in Kubernetes.
```
kubectl create -f config-map.yaml
```

- **Explain the above configuration**
- First, we need to define the target endpoints.
```
kubectl get endpoints -A
```
![endpoint](/images/3.connect/endpoint.png)
- as you can see tomcat-service is the target i want to get so i use regex to get only tomcat-service.
- The next regex I specify to only get the port that tomcat exposes the metric on. Because tomcat will expose 2 ports **8080** and **81828** and only 1 port has the metric, so to avoid confusion I only get the port that contains the metric **8182** named **prometheus**.

4. Create a Prometheus Deployment.
- Use default file **prometheus-deployment.yaml**
```
kubectl create  -f prometheus-deployment.yaml 
```
- Test connecting To Prometheus Dashboard, Exposing Prometheus as a Service NodePort.
- Create a file named **prometheus-service.yaml** and copy the following contents. We will expose Prometheus on all kubernetes node IP’s on port 30000.
```
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
  namespace: monitoring
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/port:   '9090'
spec:
  selector: 
    app: prometheus-server
  type: NodePort  
  ports:
    - port: 8080
      targetPort: 9090 
      nodePort: 30000
```

```
kubectl apply -f prometheus-service.yaml
```
- Now open port 30000, we can see that prometheus has fetched the metrics of the tomcat pods.
![result](/images/3.connect/result.png)



