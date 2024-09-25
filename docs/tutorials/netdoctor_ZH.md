---
id: netdoctor
title: 'Netdoctor'
---

# Netdoctor

## 介绍

* Kubernetes集群投入使用后集群网络可能会存在种种的连通性问题，因此我们希望可以有一个验收工具，在完成部署后检查集群的网络连通性是否正常。

* 另一方面，Kosmos是一个跨集群的解决方案，在Kosmos管理多个集群之前，需要先检查各个集群的容器网络自身是否存在问题，部署完成后也需要验证跨集群的网络是否已经被Kosmos连通。

* 出于以上两个方面，我们设计了 [NetDoctor](https://github.com/kosmos-io/netdoctor) 工具用于解决Kubernetes集群遇到的网络问题。

## 架构

![NetDoctor](../tutorials/img/netdr-arch.png)

## 先决条件

* `go` 版本 v1.15+
* `kubernetes` 版本 v1.16+

## 快速开始

### netctl工具
* NetDoctor提供配套工具`netctl`，您可以方便的通过命令行去进行Kubernetes集群的网络连通性检查。
#### 从制品库获取
````bash
wget https://github.com/kosmos-io/netdoctor/releases/download/v0.0.1/netctl-linux-amd64 
mv netctl-linux-amd64 netctl
````
#### 从源码编译
````bash
# 下载项目源码
$ git clone https://github.com/kosmos-io/netdoctor.git
# 执行后netctl会输出至./netdoctor/_output/bin/linux/amd64目录
$ make netctl
````
### netctl命令
* `netctl init`命令用于在当前目录生成网络检查需要的配置文件`config.json`，示例如下：
````bash
$ netctl init
I0205 16:27:26.258964 2765415 init.go:69] write opts success
$ cat config.json
{
 "namespace": "kosmos-system",
 "version": "v0.2.0",
 "protocol": "tcp",
 "podWaitTime": 30,
 "port": "8889",
 "maxNum": 3,
 "cmdTimeout": 10,
 "srcKubeConfig": "~/.kube/config",
 "srcImageRepository": "ghcr.io/kosmos-io"
}
````

* `netctl check`命令会读取`config.json`，然后创建一个名为`Floater`的`DaemonSet`以及相关联的一些资源，之后会获取所有的`Floater`的`IP`信息，然后依次进入到`Pod`中执行`Ping`或者`Curl`命令。需要注意的是，这个操作是并发执行的，并发度根据`config.json`中的`maxNum`参数动态变化。
````bash
$ netctl check
I0205 16:34:06.147671 2769373 check.go:61] use config from file!!!!!!
I0205 16:34:06.148619 2769373 floater.go:73] create Clusterlink floater, namespace: kosmos-system
I0205 16:34:06.157582 2769373 floater.go:83] create Clusterlink floater, apply RBAC
I0205 16:34:06.167799 2769373 floater.go:94] create Clusterlink floater, version: v0.2.0
I0205 16:34:09.178566 2769373 verify.go:79] pod: clusterlink-floater-9dzsg is ready. status: Running
I0205 16:34:09.179593 2769373 verify.go:79] pod: clusterlink-floater-cscdh is ready. status: Running
Do check... 100% [================================================================================]  [0s]
+-----+----------------+----------------+-----------+-----------+
| S/N | SRC NODE NAME  | DST NODE NAME  | TARGET IP |  RESULT   |
+-----+----------------+----------------+-----------+-----------+
|   1 | ecs-net-dr-001 | ecs-net-dr-001 | 10.0.1.86 | SUCCEEDED |
|   2 | ecs-net-dr-002 | ecs-net-dr-002 | 10.0.2.29 | SUCCEEDED |
+-----+----------------+----------------+-----------+-----------+

+-----+----------------+----------------+-----------+-----------+-------------------------------+
| S/N | SRC NODE NAME  | DST NODE NAME  | TARGET IP |  RESULT   |              LOG              |
+-----+----------------+----------------+-----------+-----------+-------------------------------+
|   1 | ecs-net-dr-002 | ecs-net-dr-001 | 10.0.1.86 | EXCEPTION |exec error: unable to upgrade  |
|   2 | ecs-net-dr-001 | ecs-net-dr-002 | 10.0.2.29 | EXCEPTION |connection: container not......|
+-----+----------------+----------------+-----------+-----------+-------------------------------+
I0205 16:34:09.280220 2769373 do.go:93] write opts success
````
* 在`check`命令执行的过程中，会有进度条显示校验进度。命令执行完成后，会打印检查结果，并将结果保存在文件`resume.json`中。
````bash
[
 {
  "Status": 0,
  "ResultStr": "exec error: unable to upgrade connection: container not found (\"floater\"), stderr: ",
  "srcNodeName": "ecs-sealos-001",
  "dstNodeName": "ecs-sealos-002",
  "targetIP": "10.0.2.29"
 },
 {
  "Status": 0,
  "ResultStr": "exec error: command terminated with exit code 7, stderr  % Total  % Received % Xferd  Average  Speed  Time  Time  Time  Current\n  Dload  Upload  Total  Spent  Left  Speed\n\r  0  0  0  0  0  0  0  0 --:--:-- --:--:-- --:--:--  0\r  0  0  0  0  0  0  0  0 --:--:-- --:--:-- --:--:--  0\ncurl: (7) Failed to connect to 10.0.0.36 port 8889 after 0 ms: Couldn't connect to server\n",
  "srcNodeName": "ecs-sealos-002",
  "dstNodeName": "ecs-sealos-001",
  "targetIP": "10.0.0.36"
 }
]
````

* 如果需要检查Kosmos集群联邦中任意两个集群之间的网络连通性，则可以在配置文件`config.json`增加参数`dstKubeConfig`和`dstImageRepository`，这样就可以检查两个集群之间网络连通性了。
````bash
$ vim config.json
{
 "namespace": "kosmos-system",
 "version": "v0.2.0",
 "protocol": "tcp",
 "podWaitTime": 30,
 "port": "8889",
 "maxNum": 3,
 "cmdTimeout": 10,
 "srcKubeConfig": "~/.kube/src-config",
 "srcImageRepository": "ghcr.io/kosmos-io"
 "dstKubeConfig": "~/.kube/dst-config",
 "dstImageRepository": "ghcr.io/kosmos-io"
}
````

* `netctl resume`命令用于复测时只检验第一次检查时有问题的集群节点。因为线上环境节点数量很多，单次检查可能会需要比较长的时间才能生成结果，所以我们希望仅对前一次检查异常的节点进行复测。`resume`命令因此被开发，该命令会读取`resume.json`文件，并对前一次异常的节点进行再次检查，我们可以重复执行此命令至没有异常的结果后再执行全量检查。
````bash
$ netctl resume
I0205 16:34:06.147671 2769373 check.go:61] use config from file!!!!!!
I0205 16:34:06.148619 2769373 floater.go:73] create Clusterlink floater, namespace: kosmos-system
I0205 16:34:06.157582 2769373 floater.go:83] create Clusterlink floater, apply RBAC
I0205 16:34:06.167799 2769373 floater.go:94] create Clusterlink floater, version: v0.2.0
I0205 16:34:09.178566 2769373 verify.go:79] pod: clusterlink-floater-9dzsg is ready. status: Running
I0205 16:34:09.179593 2769373 verify.go:79] pod: clusterlink-floater-cscdh is ready. status: Running
Do check... 100% [================================================================================]  [0s]
+-----+----------------+----------------+-----------+-----------+
| S/N | SRC NODE NAME  | DST NODE NAME  | TARGET IP |  RESULT   |
+-----+----------------+----------------+-----------+-----------+
|   1 | ecs-net-dr-002 | ecs-net-dr-001 | 10.0.1.86 | SUCCEEDED |
|   2 | ecs-net-dr-001 | ecs-net-dr-002 | 10.0.2.29 | SUCCEEDED |
+-----+----------------+----------------+-----------+-----------+
````

* `netctl clean`命令用于清理`NetDoctor`创建的所有资源。
