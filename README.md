# TiDB Deployment on Graviton2-based EKS

## Prepare ARM based docker images

For `tidb-operator`, change `busybox` and re-compile using `GOARCH=arm64 make`.</br>
Build new docker image like:</br>
```shell
docker build -t pingcap2021/tidb-operator:v1.1.14 -f Dockerfile 
```

For `tidb`,`tikv` and `pd`, use `centos:8` as base image.</br>

For `tidb-monitor-initializer`, following Jenkins pipleine to compile it. (`token` is needed, if you have compiled before, please close previous PR, repull everything following Jenkins pipeline), and put it in the Dockerfile.</br>

For `br`, use `centos:8` as base image.</br>

For `tidb-backup-manager`, change base image and `amd64` to `arm64` in Dockerfile.</br>
```
# sed -i 's/FROM.*/FROM centos:/' images/tidb-backup-manager/Dockerfile
# sed -i 's/amd64/arm64/g' images/tidb-backup-manager/Dockerfile
# GOARCH=arm64 GOOS=linux DOCKER_REPO=pingcap make backup-docker
```
And on `entrypoint.sh` line 55, insert one extra space before the right square bracket as shown below.
```shell
BACKUP_BIN=/tidb-bakcup-manager
if [[ -n "${AWS_DEFAULT_REGION}" ]];then
  EXEC_COMMAND="exec"
else
```

For the time being, we didn't build arm images for `tidb-monitor-reloader`. </br>

All the newly built temp images for ARM can be found at:
* pingcap2021/tidb-operator:v1.1.14
* pingcap2021/pd:v4.0.10
* pingcap2021/tikv:v4.0.10
* pingcap2021/tidb:v4.0.10
* pingcap2021/tidb-monitor-initializer:v4.0.10
* pingcap2021/tidb-backup-manager:v1.1.11
* pingcap2021/br:v4.0.10

## Deploy EKS cluster

### Prerequisites
1. Install Helm 3: used for deploying TiDB Operator.
2. Complete all operations in Getting started with eksctl.
3. Install kubectl.

### Create a EKS cluster and a node pool

Sample cluster.yaml for a 3-node TiDB, 3-node TiKV and one node PD cluster.
```YAML
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: benchmark-arm #replace with your cluster name
  region: ap-southeast-1 #replace with your preferred AWS region
nodeGroups:
  - name: admin
    desiredCapacity: 1
    instanceType: c6g.large
    privateNetworking: true
    labels:
      dedicated: admin
  - name: tidb-1a
    desiredCapacity: 1
    instanceType: c6g.2xlarge
    privateNetworking: true
    availabilityZones: ["ap-southeast-1a"]
    labels:
      dedicated: tidb
    taints:
      dedicated: tidb:NoSchedule
  - name: tidb-1b
    desiredCapacity: 1
    instanceType: c6g.2xlarge
    privateNetworking: true
    availabilityZones: ["ap-southeast-1b"]
    labels:
      dedicated: tidb
    taints:
      dedicated: tidb:NoSchedule
  - name: tidb-1c
    desiredCapacity: 1
    instanceType: c6g.2xlarge
    privateNetworking: true
    availabilityZones: ["ap-southeast-1c"]
    labels:
      dedicated: tidb
    taints:
      dedicated: tidb:NoSchedule
  - name: pd-1a
    desiredCapacity: 1
    instanceType: c6g.large
    privateNetworking: true
    availabilityZones: ["ap-southeast-1a"]
    labels:
      dedicated: pd
    taints:
      dedicated: pd:NoSchedule
  - name: tikv-1a
    desiredCapacity: 1
    instanceType: r6g.2xlarge
    privateNetworking: true
    availabilityZones: ["ap-southeast-1a"]
    labels:
      dedicated: tikv
    taints:
      dedicated: tikv:NoSchedule
  - name: tikv-1b
    desiredCapacity: 1
    instanceType: r6g.2xlarge
    privateNetworking: true
    availabilityZones: ["ap-southeast-1b"]
    labels:
      dedicated: tikv
    taints:
      dedicated: tikv:NoSchedule
  - name: tikv-1c
    desiredCapacity: 1
    instanceType: r6g.2xlarge
    privateNetworking: true
    availabilityZones: ["ap-southeast-1c"]
    labels:
      dedicated: tikv
    taints:
      dedicated: tikv:NoSchedule
```

Execute the following command to create the cluster:
```shell
eksctl create cluster -f cluster.yaml
```

## Deploy TiDB Operator

Follow the steps described in [Deploy TiDB Operator](https://docs.pingcap.com/tidb-in-kubernetes/stable/get-started#deploy-tidb-operator)

For step 3 of installing the TiDB Operator, use the arm images as follows:
```shell
helm install --namespace tidb-admin tidb-operator pingcap/tidb-operator --version v1.1.10 \
    --set operatorImage=pingcap2021/tidb-operator:v1.1.14
```