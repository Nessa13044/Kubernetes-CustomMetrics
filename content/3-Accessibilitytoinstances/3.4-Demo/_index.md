---
title : "Demo"
date : "`r Sys.Date()`"
weight : 4
chapter : false
pre : " <b> 3.4. </b> "
---

- Use this command to create a pod named "loadgenerator" and continuously send HTTP requests to an IP address (here tomcat's service 10.98.29.31:8080) to simulate increased load on the service at that address.
```
kubectl run -it --rm --restart=Never loadgenerator --image=busybox -- sh -c "while true; do wget -O - -q http://10.98.29.31:8080; done"
```
{{% notice note %}}
Note: Replace with your tomcat services.
{{% /notice %}}

- Open another terminal to see the whole HPA scaling process.
```
kubectl get hpa -A -w
```

![endpoint](/images/3.connect/hpa1.png)
- As you can see, the larger the number of requests (more than 30k here), the number of pods automatically expanded to 3 copies as we configured.
![endpoint](/images/3.connect/c2.png)
![endpoint](/images/3.connect/check.png)
