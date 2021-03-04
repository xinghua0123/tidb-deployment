# TiDB Deployment on Graviton2-based EKS

## Prepare ARM based docker images

For `tidb-operator`,  re-compile using `GOARCH=arm64 make`.</br>
Build new docker image like:</br>
```shell
docker build -t pingcap2021/tidb-operator:v1.1.14 -f Dockerfile 
```

For `tidb`,`tikv` and `pd`, use `centos:8` as base image.</br>

For `tidb-monitor-initializer`, following Jenkins pipleine to compile it. (`token` is needed, if you have compiled before, please close previous PR, repull everything following Jenkins pipeline), and put it in the Dockerfile.</br>

For `br`, use `centos:8` as base image.</br>

For `tidb-backup-manager`, change base image and `amd64` to `arm64` in Dockerfile.</br>
```
# sed -i 's/FROM.*/FROM centos:8/' images/tidb-backup-manager/Dockerfile
# sed -i 's/amd64/arm64/g' images/tidb-backup-manager/Dockerfile
# GOARCH=arm64 GOOS=linux DOCKER_REPO=pingcap make backup-docker
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
    instanceType: c6g.large # use Graviton2 powered EC2 instances
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

## Deploy TiDB Cluster

### TiDB Cluster
Download sample `TidbCluster` and `TidbMonitor`CR YAML:
```shell
curl -O https://raw.githubusercontent.com/pingcap/tidb-operator/master/examples/aws/tidb-cluster.yaml && \
curl -O https://raw.githubusercontent.com/pingcap/tidb-operator/master/examples/aws/tidb-monitor.yaml
```

Edit configurations to use the arm images.
`TidbCluster` YAML
```YAML
apiVersion: pingcap.com/v1alpha1
kind: TidbCluster
metadata:
  name: arm-test
  namespace: tidb-cluster
spec:
  version: v4.0.10
  timezone: UTC
  configUpdateStrategy: RollingUpdate
  pvReclaimPolicy: Retain
  schedulerName: tidb-scheduler
  enableDynamicConfiguration: true
  pd:
    baseImage: pingcap2021/pd # use the arm image
    replicas: 1
    requests:
      storage: "50Gi"
    config:
      log:
        level: info
      replication:
        location-labels:
        - zone
        max-replicas: 1
    nodeSelector:
      dedicated: pd
    tolerations:
    - effect: NoSchedule
      key: dedicated
      operator: Equal
      value: pd
  tikv:
    baseImage: pingcap2021/tikv # use the arm image
    replicas: 3
    requests:
      storage: "1000Gi"
    config: {}
    nodeSelector:
      dedicated: tikv
    tolerations:
    - effect: NoSchedule
      key: dedicated
      operator: Equal
      value: tikv
  tidb:
    baseImage: pingcap2021/tidb # use the arm image
    replicas: 3
    service:
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: 'true'
        # To expose the LB to the internet
        # service.beta.kubernetes.io/aws-load-balancer-internal: '0.0.0.0/0'
        service.beta.kubernetes.io/aws-load-balancer-type: nlb 
      exposeStatus: true
      externalTrafficPolicy: Local
      type: LoadBalancer
    config:
      log:
        level: info
      performance:
        max-procs: 0
        tcp-keep-alive: true
    annotations:
      tidb.pingcap.com/sysctl-init: "true"
    podSecurityContext:
      sysctls:
      - name: net.ipv4.tcp_keepalive_time
        value: "300"
      - name: net.ipv4.tcp_keepalive_intvl
        value: "75"
      - name: net.core.somaxconn
        value: "32768"
    separateSlowLog: true
    nodeSelector:
      dedicated: tidb
    tolerations:
    - effect: NoSchedule
      key: dedicated
      operator: Equal
      value: tidb
```

After you have deployed `TidbCluster` YAML, you will see something like below:
```shell
kubectl get pods -n tidb-cluster
NAME                                  READY   STATUS     RESTARTS   AGE
arm-test-discovery-545b84f7b6-4ch4x   1/1     Running    0          27m
arm-test-pd-0                         1/1     Running    0          27m
arm-test-tidb-0                       2/2     Running    0          25m
arm-test-tidb-1                       2/2     Running    0          25m
arm-test-tidb-2                       2/2     Running    0          25m
arm-test-tikv-0                       1/1     Running    0          26m
arm-test-tikv-1                       1/1     Running    0          26m
arm-test-tikv-2                       1/1     Running    0          26m
```

Until now, TiDB cluster is up and running. You can try to connect to TiDB cluster using:
```shell
mysql -h ${tidb-nlb-dnsname} -P 4000 -u root
```
where `${tidb-nlb-dnsname}` is the LoadBalancer domain name of the TiDB service under `tidb-cluster` namespace.

### TiDB Monitor
`TidbMonitor` YAML:
```YAML
apiVersion: pingcap.com/v1alpha1
kind: TidbMonitor
metadata:
  name: ron-monitor
spec:
  alertmanagerURL: ""
  annotations: {}
  clusters:
  - name: arm-test
  grafana:
    baseImage: grafana/grafana
    envs:
      # Configure Grafana using environment variables except GF_PATHS_DATA, GF_SECURITY_ADMIN_USER and GF_SECURITY_ADMIN_PASSWORD
      # Ref https://grafana.com/docs/installation/configuration/#using-environment-variables
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_NAME: "Main Org."
      GF_AUTH_ANONYMOUS_ORG_ROLE: "Viewer"
      # if grafana is running behind a reverse proxy with subpath http://foo.bar/grafana
      # GF_SERVER_DOMAIN: foo.bar
      # GF_SERVER_ROOT_URL: "%(protocol)s://%(domain)s/grafana/"
    imagePullPolicy: IfNotPresent
    logLevel: info
    password: admin
    service:
      portName: http-grafana
      type: LoadBalancer
    username: admin
    version: 6.1.6
  imagePullPolicy: IfNotPresent
  initializer:
    baseImage: pingcap2021/tidb-monitor-initializer # use the arm image
    imagePullPolicy: IfNotPresent
    version: v4.0.10
  kubePrometheusURL: ""
  persistent: true
  prometheus:
    baseImage: prom/prometheus
    imagePullPolicy: IfNotPresent
    logLevel: info
    reserveDays: 12
    service:
      portName: http-prometheus
      type: NodePort
    version: v2.18.1
  reloader:
    baseImage: pingcap/tidb-monitor-reloader 
    imagePullPolicy: IfNotPresent
    service:
      portName: tcp-reloader
      type: NodePort
    version: v1.0.1
  storage: 100Gi
```

After you deployed the monitor, the pod will be in error state since the `tidb-monitor-reloader` doesn't have a arm specific image as of now.</br>

Instead, get the monitor deployment using command:
```shell
kubectl get deploy ${monitor-deployment-name} -n tidb-cluster -o yaml
```

Remove the `tidb-monitor-reloader` container from the deployment YAML and also rename the deployment.

Delete existing deployment and deploy the new monitor deployment.

Once the monitor deployment is done successfully, access the Grafana dashboard from the LoadBalancer domain name of the grafana service under `tidb-cluster` namespace.