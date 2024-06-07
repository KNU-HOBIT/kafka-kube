# Kafka Cluster Deployment with Strimzi

This repository contains Kubernetes manifests to deploy a Kafka cluster using Strimzi with node pools and specific storage configurations. The Kafka cluster is set up to use KRaft mode and is configured with persistent storage using rook-cephfs.

## Pre-requisites

1. A running Kubernetes cluster with kubectl configured.
2. Rook Ceph installed and configured for storage.

## Installation


### 1. Strimzi Kafka Operator installed on the cluster.

```sh
# 네임스페이스 생성
kubectl create namespace kafka

# Add the Strimzi Helm Chart repository:
helm repo add strimzi https://strimzi.io/charts/

# install the chart with the release name my-release:
helm install kafka-operator strimzi/strimzi-kafka-operator --version 0.28.0 -n kafka

# 배포한 리소스 확인 : CRDs
kubectl get crd
```
```
NAME                                  CREATED AT
kafkabridges.kafka.strimzi.io         2022-05-15T14:00:21Z
kafkaconnectors.kafka.strimzi.io      2022-05-15T14:00:22Z
kafkaconnects.kafka.strimzi.io        2022-05-15T14:00:21Z
kafkamirrormaker2s.kafka.strimzi.io   2022-05-15T14:00:22Z
kafkamirrormakers.kafka.strimzi.io    2022-05-15T14:00:21Z
kafkarebalances.kafka.strimzi.io      2022-05-15T14:00:22Z
kafkas.kafka.strimzi.io               2022-05-15T14:00:21Z
kafkatopics.kafka.strimzi.io          2022-05-15T14:00:21Z
kafkausers.kafka.strimzi.io           2022-05-15T14:00:21Z
strimzipodsets.core.strimzi.io        2022-05-15T14:00:21Z
```

```sh
# 배포한 리소스 확인 : Operator 디플로이먼트(파드)
kubectl get deploy,pod -n kafka
```

```
NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/strimzi-cluster-operator   1/1     1            1           3m37s

NAME                                            READY   STATUS    RESTARTS   AGE
pod/strimzi-cluster-operator-7599bc57cb-2zlrt   1/1     Running   0          3m37s
```

### 2. 카프카 클러스터 생성


```sh
git clone https://github.com/KNU-HOBIT/kafka-strimzi-cluster.git
cd kafka-strimzi-cluster

# 카프카 클러스터 배포
kubectl apply -f kafka-persistent.yaml -n kafka

# 배포된 리소스 확인
kubectl get kafka -n kafka
```
```
NAME         DESIRED KAFKA REPLICAS   DESIRED ZK REPLICAS   READY   METADATA STATE   WARNINGS
my-cluster   3                                              True    KRaft            True
```


```sh
# 배포된 리소스 확인 : 배포된 파드 생성 확인
kubectl get pod -n kafka -l app.kubernetes.io/instance=my-cluster
```
```
NAME                                          READY   STATUS    RESTARTS       AGE
my-cluster-dual-role-2-1                      1/1     Running   0              43d
my-cluster-kafka-exporter-7f757c6459-gjbbt    1/1     Running   3 (16d ago)    43d
my-cluster-entity-operator-7f6cd75856-dzxld   2/2     Running   13 (16d ago)   43d
my-cluster-dual-role-3-2                      1/1     Running   1 (16d ago)    43d
my-cluster-dual-role-1-0                      1/1     Running   8 (5d8h ago)   43d
```

```sh
# 배포된 리소스 확인 : 서비스 Service(Headless) 생성 확인
kubectl get svc -n kafka
```
```
NAME                                  TYPE      ...  PORT(S)                               AGE
my-cluster-kafka-bootstrap            ClusterIP ...  9091/TCP,9092/TCP                     43d
my-cluster-kafka-brokers              ClusterIP ...  9090/TCP,9091/TCP,8443/TCP,9092/TCP   43d
my-cluster-kafka-nodeport-bootstrap   NodePort  ...  9093:32100/TCP                        43d
my-cluster-dual-role-1-nodeport-0     NodePort  ...  9093:32000/TCP                        43d
my-cluster-dual-role-2-nodeport-1     NodePort  ...  9093:32001/TCP                        43d
my-cluster-dual-role-3-nodeport-2     NodePort  ...  9093:32002/TCP                        43d
```

```sh
# kafka 클러스터 Listeners 정보 확인
kubectl get kafka -n kafka my-cluster -o jsonpath={.status} | jq
```
```json
"listeners": [
    {
      "addresses": [
        {
          "host": "my-cluster-kafka-bootstrap.kafka.svc",
          "port": 9092
        }
      ],
      "bootstrapServers": "my-cluster-kafka-bootstrap.kafka.svc:9092",
      "name": "plain"
    },
    {
      "addresses": [
        {
          "host": "<node3-IP>",
          "port": 32100
        },
        {
          "host": "<node2-IP>",
          "port": 32100
        },
        {
          "host": "<node1-IP>",
          "port": 32100
        }
      ],
      "bootstrapServers": "<node1-IP>:32100,<node2-IP>:32100,<node3-IP>:32100",
      "name": "nodeport"
    }
  ],
```


### 3. 토픽 생성
```sh
# 토픽 Topic 생성 : 파티션 2개, 리플리케이션 3개
cat << EOF | kubectl create -n kafka -f -
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: my-topic
  labels:
    strimzi.io/cluster: "my-cluster"
spec:
  partitions: 2
  replicas: 3
  #config:
  #  retention.ms: 7200000 -> 메세지가 유지되는 시간(ms)
  #  segment.bytes: 1073741824
EOF
```
```sh
# 토픽 Topic 생성 확인
kubectl get kafkatopics -n kafka
```
```
NAME              CLUSTER      PARTITIONS   REPLICATION FACTOR   READY
transport-203     my-cluster   1            1                    True
transport-355     my-cluster   1            1                    True
transport-305     my-cluster   1            1                    True
```