## 安装es
```sh
# GitHub仓库地址 
https://github.com/elastic/helm-charts


# 添加elastic的chart仓库
helm repo add elastic https://helm.elastic.co

# 查看elasticsearch最近版本
helm search repo elastic/elasticsearch

# 下载最新版本
helm pull elastic/elasticsearch

# 编辑values文件

# 安装master节点
helm install elasticsearch -f es-master-value.yaml ./elasticsearch -n es

# 安装data节点
helm install elasticsearch-data -f es-data-value.yaml ./elasticsearch -n es

# 安装client节点
helm install elasticsearch-client -f es-client-value.yaml ./elasticsearch -n es

# 安装kibana
helm pull elastic/kibana
helm install kibana ./kibana -n es

```

## 利用开源组件Log-pilot搭建Kubernetes日志解决方案

参考阿里文档

https://help.aliyun.com/document_detail/86552.html

部署log-pilot的yaml文件
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-pilot
  labels:
    app: log-pilot
  # 设置期望部署的namespace。
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: log-pilot
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: log-pilot
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      # 是否允许部署到Master节点上tolerations。
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: log-pilot
        # 版本请参考https://github.com/AliyunContainerService/log-pilot/releases。
        image: registry.cn-hangzhou.aliyuncs.com/acs/log-pilot:0.9.6-filebeat
        resources:
          limits:
            memory: 500Mi
          requests:
            cpu: 200m
            memory: 200Mi
        env:
          - name: "NODE_NAME"
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          ## 1. 对接kafka集群
          - name: LOGGING_OUTPUT
            value: kafka
          # kafka 要和 log-pilot 在一个命名空间
          - name: KAFKA_BROKERS
            value: bootstrap:9092
          ## 2. 直接对接es集群
          # - name: "LOGGING_OUTPUT"
          #   value: "elasticsearch"
          # # 请确保集群到ES网络可达。
          # - name: "ELASTICSEARCH_HOSTS"
          #   value: "{es_endpoint}:{es_port}"
          # # 配置ES访问权限。
          # - name: "ELASTICSEARCH_USER"
          #   value: "{es_username}"
          # - name: "ELASTICSEARCH_PASSWORD"
          #   value: "{es_password}"
        volumeMounts:
        - name: sock
          mountPath: /var/run/docker.sock
        - name: root
          mountPath: /host
          readOnly: true
        - name: varlib
          mountPath: /var/lib/filebeat
        - name: varlog
          mountPath: /var/log/filebeat
        - name: localtime
          mountPath: /etc/localtime
          readOnly: true
        livenessProbe:
          failureThreshold: 3
          exec:
            command:
            - /pilot/healthz
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 2
        securityContext:
          capabilities:
            add:
            - SYS_ADMIN
      terminationGracePeriodSeconds: 30
      volumes:
      - name: sock
        hostPath:
          path: /var/run/docker.sock
      - name: root
        hostPath:
          path: /
      - name: varlib
        hostPath:
          path: /var/lib/filebeat
          type: DirectoryOrCreate
      - name: varlog
        hostPath:
          path: /var/log/filebeat
          type: DirectoryOrCreate
      - name: localtime
        hostPath:
          path: /etc/localtime
```

被收集的pod设置举例
```sh
aliyun_logs_logs-catalina=stdout表示要收集容器的stdout日志。
aliyun_logs_logs-access=/usr/local/tomcat/logs/catalina.*.log表示要收集容器内/usr/local/tomcat/logs/目录下所有名字匹配catalina.*.log的文件日志。
在本方案的Elasticsearch场景下，环境变量中的$name表示Index，本例中$name即是logs-catalina和logs-access
```
```yaml
    env:
    # 1、stdout为约定关键字，表示采集标准输出日志。
    # 2、配置标准输出日志采集到ES的catalina索引下。
    - name: aliyun_logs_logs-catalina
      value: "stdout"
    # 1、配置采集容器内文件日志，支持通配符。
    # 2、配置该日志采集到ES的access索引下。
    - name: aliyun_logs_logs-access
      value: "/usr/local/tomcat/logs/catalina.*.log"
    # 容器内文件日志路径需要配置emptyDir。

```

## 安装logstash
```sh
# 安装kibana
helm pull elastic/logstash
helm install logstash ./logstash -n es

```

## 安装zookeeper
```sh

# helm search repo bitnami/zookeeper 
NAME                    CHART VERSION   APP VERSION     DESCRIPTION                                       
bitnami/zookeeper       6.7.0           3.7.0           A centralized service for maintaining configura...

# 下载chart包, 编辑 values.yaml 文件
helm pull bitnami/zookeeper

# 部署
helm install zk -n ops ./zookeeper
WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /Users/shenxiaoshuai/Desktop/kkb/baiwan.conf
WARNING: Kubernetes configuration file is world-readable. This is insecure. Location: /Users/shenxiaoshuai/Desktop/kkb/baiwan.conf
NAME: zk
LAST DEPLOYED: Thu Apr 15 18:13:07 2021
NAMESPACE: ops
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
** Please be patient while the chart is being deployed **

ZooKeeper can be accessed via port 2181 on the following DNS name from within your cluster:

    zk-zookeeper.ops.svc.cluster.local

To connect to your ZooKeeper server run the following commands:

    export POD_NAME=$(kubectl get pods --namespace ops -l "app.kubernetes.io/name=zookeeper,app.kubernetes.io/instance=zk,app.kubernetes.io/component=zookeeper" -o jsonpath="{.items[0].metadata.name}")
    kubectl exec -it $POD_NAME -- zkCli.sh

To connect to your ZooKeeper server from outside the cluster execute the following commands:

    kubectl port-forward --namespace ops svc/zk-zookeeper 2181:2181 &
    zkCli.sh 127.0.0.1:2181
```

## 安装kafka
```sh
helm search repo bitnami/kafka
NAME            CHART VERSION   APP VERSION     DESCRIPTION                                      
bitnami/kafka   12.17.3         2.7.0           Apache Kafka is a distributed streaming platform.

# 下载chart包, 编辑 values.yaml 文件
helm pull bitnami/kafka

# 部署
helm install kafka -n ops ./kafka
NAME: kafka
LAST DEPLOYED: Thu Apr 15 19:23:36 2021
NAMESPACE: ops
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
** Please be patient while the chart is being deployed **

Kafka can be accessed by consumers via port 9092 on the following DNS name from within your cluster:

    kafka.ops.svc.cluster.local

Each Kafka broker can be accessed by producers via port 9092 on the following DNS name(s) from within your cluster:

    kafka-0.kafka-headless.ops.svc.cluster.local:9092
    kafka-1.kafka-headless.ops.svc.cluster.local:9092
    kafka-2.kafka-headless.ops.svc.cluster.local:9092

To create a pod that you can use as a Kafka client run the following commands:

    kubectl run kafka-client --restart='Never' --image docker.io/bitnami/kafka:2.7.0-debian-10-r118 --namespace ops --command -- sleep infinity
    kubectl exec --tty -i kafka-client --namespace ops -- bash

    PRODUCER:
        kafka-console-producer.sh \
            --broker-list kafka-0.kafka-headless.ops.svc.cluster.local:9092,kafka-1.kafka-headless.ops.svc.cluster.local:9092,kafka-2.kafka-headless.ops.svc.cluster.local:9092 \
            --topic test

    CONSUMER:
        kafka-console-consumer.sh \
            --bootstrap-server kafka.ops.svc.cluster.local:9092 \
            --topic test \
            --from-beginning
```

## 安装kafkaManager
```sh
helm search repo stable/kafka-manager
NAME                    CHART VERSION   APP VERSION     DESCRIPTION                                   
stable/kafka-manager    2.3.5           1.3.3.22        DEPRECATED - A tool for managing Apache Kafka.

helm install kafka-manager -n ops ./kafka-manager
NAME: kafka-manager
LAST DEPLOYED: Thu Apr 15 20:07:35 2021
NAMESPACE: ops
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace ops -l "app=kafka-manager,release=kafka-manager" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl port-forward $POD_NAME 8080:9000

```