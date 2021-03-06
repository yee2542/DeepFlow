# Deployments

- fluentsearch-fe `(port: 80)`
- fluentsearch-bff `(port: 3000)`
- fluentsearch-storage `(port: 3000)`
- fluentsearch-admission `(port: 3000)`
- fluentsearch-ml `(port: 8080)`

# Services

- fluentsearch-fe-service `(port: 3000)`
- fluentsearch-bff-service `(port: 5000)`
- fluentsearch-storage-service `(port: 3000)`
- fluentsearch-admission-service `(port: 3000)`
- kubernetes-dashboard `(port: 443)`
- fluentsearch-mongodb-mongodb-sharded `(port: 27017)`
- rabbitmq `(port:5672,15672)`
- fluentsearch-ml-service (port:8080)

# Ingress

- kubernetes-dashboard `(host: dashboard.fluentsearch.local)`
- fluentsearch-bff-ingress `(host: api.fluentsearch.ml)`
- fluentsearch-fe-ingress `(host: fluentsearch.ml)`
- fluentsearch-storage-ingress `(host: storage.fluentsearch.ml)`
- fluentsearch-api-federation-ingress `(host: federation.fluentsearch.ml)`

# API Federation

```sh
$ kubectl create namespace fluentsearch-api-federation
$ kubectl create -f ./api-federation
```

# ML

```sh
$ kubectl create namespace fluentsearch-ml
$ kubectl create -f ./ml
```

# Admission

```sh
$ kubectl create namespace fluentsearch-admission
$ kubectl create -f ./admission
```

# FE

```sh
$ kubectl create namespace fluentsearch-fe
$ kubectl create -f ./fe
```

# BFF

```sh
$ kubectl create namespace fluentsearch-bff
$ kubectl create -f ./bff
```

# Storage

```sh
$ kubectl create namespace fluentsearch-storage
$ kubectl create -f ./storage
```

# Dashboard

setup dashboard to controll the master

```sh
# install
$ kubectl apply -f dashboard

# get token key
$ kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"
```

# RabbitMQ

## Setup

add helm repo

```bash
$ helm repo add bitnami https://charts.bitnami.com/bitnami
```

create namespace

```bash
kubectl apply -f ./rabbitmq/namespace.yml
```

release

```bash
$ helm install -f ./rabbitmq/values.yml rabbitmq -n fluentsearch-mq bitnami/rabbitmq

```

## Access

RabbitMQ AMQP port:

```bash
kubectl port-forward --namespace fluentsearch-mq svc/rabbitmq 5672:5672
```

RabbitMQ Management interface

```bash
kubectl port-forward --namespace fluentsearch-mq svc/rabbitmq 15672:15672
```

## Config

| config        | value                |
| ------------- | :------------------- |
| namespaces    | fluentsearch-mq      |
| server        | 1                    |
| volume        | 1                    |
| volume size   | 2Gi                  |
| storage-class | do-block-storage     |
| user          | root                 |
| password      | FluentSearchRabbitMQ |

# Minio

## Prerequisite

- krew [install guide](https://krew.sigs.k8s.io/docs/user-guide/setup/install/)

## Setup

create operator

```sh
$ kubectl minio init
```

create tenant

```sh
$ kubectl create ns minio-tenant-1
$ kubectl minio tenant create minio-tenant-1 \
      --servers 2                             \
      --volumes 4                            \
      --capacity 100Gi                         \
      --namespace minio-tenant-1              \
      --storage-class do-block-storage
```

config values

| config        | value                  |
| ------------- | :--------------------- |
| namespaces    | minio-tenant-1         |
| operator      | 1 (ns: minio-operator) |
| server        | 2                      |
| volume        | 4                      |
| capacity      | 100Gi                  |
| storage-class | do-block-storage       |
| user          | root                   |
| password      | Fluent$earch@Minio     |

## Access

operator access

```sh
$ kubectl minio proxy -n minio-operator
# $ kubectl port-forward svc/console 9443:9443 -n minio-operator #
$ kubectl port-forward svc/console 9090:9090 -n minio-operator
```

tenant access

```sh
$ kubectl port-forward svc/minio 9000:443 -n minio-tenant-1
$ kubectl port-forward svc/minio-tenant-1-console 9443:9443 -n minio-tenant-1
```

## Users

| username | password           |
| -------- | :----------------- |
| admin    | (generated)        |
| root     | Fluent$earch@Minio |

# MongoDB

## Setup

install via `helm`

```sh
$ helm repo add bitnami https://charts.bitnami.com/bitnami

# local
$ helm install -f mongodb/values.local.yml fluentsearch-mongodb bitnami/mongodb-sharded -n fluentsearch-mongo-db

# production
$ helm install -f mongodb/values.yml fluentsearch-mongodb bitnami/mongodb-sharded -n fluentsearch-mongo-db
```

config values

| config             | value                |
| ------------------ | :------------------- |
| shards (data node) | 3                    |
| shards (replica)   | 2                    |
| arbiter            | 1                    |
| config server      | 2                    |
| mongos             | 2                    |
| replica set key    | fluentsearch         |
| root password      | Fluent$earch@MongoDB |

connect via port-forward

```sh
# connect to mongos
$ kubectl port-forward -n fluentsearch-mongo-db svc/fluentsearch-mongodb 27017:27017 &
    mongo --host 127.0.0.1 --authenticationDatabase admin -p ${MONGODB_ROOT_PASSWORD}

# port forward
$ kubectl port-forward -n fluentsearch-mongo-db svc/fluentsearch-mongodb 27017:27017

# uninstall
$ helm delete fluentsearch-mongodb -n fluentsearch-mongo-db

# proxy
kubectl port-forward --namespace fluentsearch-mongo-db svc/fluentsearch-mongodb-mongodb-sharded 27017:27017
```

refs: https://github.com/bitnami/charts/tree/master/bitnami/mongodb-sharded

## sharding enable

```sh

# enable DB
$ sh.enableSharding("${DB_NAME}")
$ sh.enableSharding("bff")

# enable collection
$ sh.shardCollection( "${COLLECTION_NAME}", { "_id" : "hashed" } )
$ sh.shardCollection( "bff-test.test", { "_id" : "hashed" } )
```

# Local Development

connect to service via minikube tunnel

```sh
$ minikube service --url $service

$ minikube service --url fluentsearch-fe-service
```

port forwarding

```sh
$ kubectl port-forward ${type/service} ${port} --addreess 0.0.0.0

$ kubectl port-forward svc/fluentsearch-fe-service 80 --address 0.0.0.0
```

# Linkerd Injection

## Setup

install linkerd

```sh
# check k8s environment
$ linkerd check --pre

# install
$ linkerd install | kubectl apply -f -

# check deploy status
$ linkerd check
$ kubectl -n linkerd get deploy
```

## Injection

inject linkerd to pods

```sh
# inject all service in fluentsearch namspace
$ kubectl get deploy -n fluentsearch -o yaml | linkerd inject - | kubectl apply -f -

# inject linkerd via folder target
$ linkerd inject ${dir} | kubectl apply -f -

# e.g. inject fe
$ linkerd inject fe | kubectl apply -f -
```

expose service after injection

```sh
$ kubectl -n default port-forward ${type/service} ${port} --address=0.0.0.0

$ kubectl -n fluentsearch port-forward svc/fluentsearch-fe-service 80 --address=0.0.0.0
```

the Linkerd dashboard by running

```sh
$ linkerd dashboard &
```

# Archive

deprecated document

## Services

- rook-ceph-mgr-dashboard-external `(port: 8443, nodePort: 30010)`

## Ceph

setup Ceph Operator for provide a storage class

```sh
# setup
$ kubectl create -f ceph/crds.yaml -f ceph/common.yaml -f ceph/operator.yaml

# check operator
$ kubectl -n rook-ceph get pod
```

after setup Ceph operator and `rook-ceph-operator` is running, now can create the Ceph cluster

```sh
# local on minikube
$ kubectl create -f ceph/cluster.local.yaml

# production
$ kubectl create -f ceph/cluster.yaml

# check all pods are running
$ kubectl -n rook-ceph get pod
```

create first object storage

```sh
$ kubectl create ceph/object.yaml
```

moniotring mon, osd via ceph-dashboard.

```sh
# get admin password
$ kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo

# expose via kube proxy
$ kubectl proxy

# links
http://localhost:8001/api/v1/namespaces/rook-ceph/services/https:rook-ceph-mgr-dashboard:https-dashboard/proxy/
```
