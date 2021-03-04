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
