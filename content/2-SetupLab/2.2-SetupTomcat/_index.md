---
title : "Setup Tomcat"
date : "`r Sys.Date()`"
weight : 2
chapter : false
pre : " <b> 2.2 </b> "
---

{{% notice info %}}
You must create your own Docker Hub account to use as a place to store images for kubernetes. 
{{% /notice %}}

- in this lab i will use tomcat metrics to condition scaling in kubernetes and to get that information i will use **JMX Exporter**
- **What's JMX Exporter**, Prometheus JMX Exporter is also a Java agent, which is capable of accessing the MBean server to access the data and transform that data into the Prometheus metric format. Prometheus then scrapes the metrics from the JMX Exporter's default metric storage path, which is /metrics.
- If you are curious, I have written a tomcat setup file, you can run it on your linux machine to see what information JMX Exporter gets from tomcat.
```
#!/bin/bash
install_jdk() {
    wget https://download.java.net/java/ga/jdk11/openjdk-11_linux-x64_bin.tar.gz
    tar xzvf openjdk-11_linux-x64_bin.tar.gz
    mv jdk-11 /opt/
    update-alternatives --install /usr/bin/java java /opt/jdk-11/bin/java 1
    update-alternatives --install /usr/bin/javac javac /opt/jdk-11/bin/javac 1
    
}

install_tomcat() {

    mkdir -p /opt/tomcat     
    wget -c https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.62/bin/apache-tomcat-9.0.62.tar.gz 
    tar xzvf apache-tomcat-9.0.62.tar.gz -C /opt/tomcat/ --strip-components=1

    groupadd tomcat
    useradd --no-create-home --shell /bin/false tomcat -g tomcat

    chown -R tomcat:tomcat /opt/tomcat/
    
    cat <<EOF > /opt/tomcat/prometheus.yml
---
lowercaseOutputLabelNames: true
lowercaseOutputName: true
rules:
- pattern: 'Catalina<type=GlobalRequestProcessor, name=\"(\w+-\w+)-(\d+)\"><>(\w+):'
  name: tomcat_\$3_total
  labels:
    port: "\$2"
    protocol: "\$1"
  help: Tomcat global \$3
  type: COUNTER
- pattern: 'Catalina<j2eeType=Servlet, WebModule=//([-a-zA-Z0-9+&@#/%?=~_|!:.,;]*[-a-zA-Z0-9+&@#/%=~_|]), name=([-a-zA-Z0-9+/$%~_-|!.]*), J2EEApplication=none, J2EEServer=none><>(requestCount|maxTime|processingTime|errorCount):'
  name: tomcat_servlet_\$3_total
  labels:
    module: "\$1"
    servlet: "\$2"
  help: Tomcat servlet \$3 total
  type: COUNTER
- pattern: 'Catalina<type=ThreadPool, name="(\w+-\w+)-(\d+)"><>(currentThreadCount|currentThreadsBusy|keepAliveCount|pollerThreadCount|connectionCount):'
  name: tomcat_threadpool_\$3
  labels:
    port: "\$2"
    protocol: "\$1"
  help: Tomcat threadpool \$3
  type: GAUGE
- pattern: 'Catalina<type=Manager, host=([-a-zA-Z0-9+&@#/%?=~_|!:.,;]*[-a-zA-Z0-9+&@#/%=~_|]), context=([-a-zA-Z0-9+/$%~_-|!.]*)><>(processingTime|sessionCounter|rejectedSessions|expiredSessions):'
  name: tomcat_session_\$3_total
  labels:
    context: "\$2"
    host: "\$1"
  help: Tomcat session \$3 total
  type: COUNTER
- pattern: 'java.lang<type=OperatingSystem><>(committed_virtual_memory|free_physical_memory|free_swap_space|total_physical_memory|total_swap_space)_size:'
  name: os_\$1_bytes
  type: GAUGE
  attrNameSnakeCase: true
- pattern: 'java.lang<type=OperatingSystem><>((?!process_cpu_time)\w+):'
  name: os_\$1
  type: GAUGE
  attrNameSnakeCase: true

EOF

    cat <<EOF > /opt/tomcat/bin/setenv.sh
CATALINA_OPTS="\$CATALINA_OPTS -javaagent:/opt/jmx_prometheus_javaagent-0.19.0.jar=8182:/opt/tomcat/prometheus.yml"
Environment=JAVA_HOME=/opt/jdk-11
export JAVA_HOME=/opt/jdk-11
EOF

    wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.19.0/jmx_prometheus_javaagent-0.19.0.jar
    mv jmx_prometheus_javaagent-0.19.0.jar /opt/


}
install_jdk
install_tomcat
```
- Run this command to start tomcat server.
```
/opt/tomcat/bin/startup.sh
```
{{% notice info %}}
Tomcat run on port 8080 and metrics expose on port 8182. 
{{% /notice %}}

- Metrics will look like this:
![jmx](/images/2.prerequisite/jmx.png)

#### Create Tomcat Images.
- Your machine has Docker installed.
- Next, execute the command below to create the directory and Dockerfile.
```
mkdir tomcat
cd tomcat
#Prepare the necessary files. Versions may vary depending on your purpose.
wget https://download.java.net/java/ga/jdk11/openjdk-11_linux-x64_bin.tar.gz
wget -c https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.62/bin/apache-tomcat-9.0.62.tar.gz
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.19.0/jmx_prometheus_javaagent-0.19.0.jar
touch Dockerfile
```
- Add the lines below to Dockerfile
```
FROM ubuntu:20.04
RUN apt-get update && apt-get install -y tar \
    && rm -rf /var/lib/apt/lists/*

COPY openjdk-11_linux-x64_bin.tar.gz /tmp/
RUN mkdir -p /opt/jdk-11 \
    && tar xzvf /tmp/openjdk-11_linux-x64_bin.tar.gz -C /opt/jdk-11/ --strip-components=1 \
    && update-alternatives --install /usr/bin/java java /opt/jdk-11/bin/java 1 \
    && update-alternatives --install /usr/bin/javac javac /opt/jdk-11/bin/javac 1 

COPY apache-tomcat-9.0.62.tar.gz /tmp/ 
RUN mkdir -p /opt/tomcat \
    && tar xzvf /tmp/apache-tomcat-9.0.62.tar.gz -C /opt/tomcat/ --strip-components=1 

RUN groupadd tomcat \
    && useradd --no-create-home --shell /bin/false tomcat -g tomcat \
    && chown -R tomcat:tomcat /opt/tomcat/

COPY prometheus.yml /opt/tomcat/
COPY setenv.sh /opt/tomcat/bin/

COPY jmx_prometheus_javaagent-0.19.0.jar /opt/
EXPOSE 8182 8080
CMD ["/opt/tomcat/bin/catalina.sh", "run"]
```
- Create Docker image using the following command:
```
docker build -t my-tomcat-image .
```
- Image created successfully.
![images](/images/2.prerequisite/images.png)

- In your machine, login on your Docker Hub. You will be asked to enter a username and password. The password will be encrypted and stored locally.
```
docker login
```
- Tag images with names in Docker Hub.
```
#docker tag <image_id> <dockerhub_username>/<repository_name>:<tag>
docker tag 282612a56d35 nessa13044/tomcat-k8s:v1
```
- Push Image to Docker Hub.
```
docker push nessa13044/tomcat-k8s:v1
```

- check images are ready on Docker Hub.
![docker](/images/2.prerequisite/docker-1.png)

