---
id: netdoctor
title: 'Netdoctor'
---

# Netdoctor

## Introduction

After the Kubernetes cluster is put into use, the cluster network may have various connectivity problems, so we hope to have an acceptance tool to check whether the network connectivity of the cluster is normal after the deployment is completed.

On the other hand, Kosmos is a cross-cluster solution. Before Kosmos manages multiple clusters, you need to check whether there are problems with the container network of each cluster itself. After the deployment is completed, you also need to verify whether the cross-cluster network has been connected by Kosmos.

For the above two aspects, we designed the [NetDoctor](https://github.com/kosmos-io/netdoctor) tool to solve network problems encountered by Kubernetes clusters.

## Architecture

![NetDoctor](../tutorials/img/netdr-arch.png)

## Prerequisites

* `go` version v1.15+
* `kubernetes` version v1.16+

## Quick Start

### Netctl
NetDoctor provides a supporting tool `netctl`, which allows you to easily check the network connectivity of the Kubernetes cluster through the command line.
#### From artifact
````bash
wget https://github.com/kosmos-io/netdoctor/releases/download/v0.0.1/netctl-linux-amd64 
mv netctl-linux-amd64 netctl
````
#### From source code
````bash
# Download project source code
$ git clone https://github.com/kosmos-io/netdoctor.git
# After execution, netctl will output to the ./netdoctor/_output/bin/linux/amd64 directory
$ make netctl
````

### Command
* `netctl init` command is used to generate the configuration file `config.json` required for network inspection in the current directory. The example is as follows:
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

* `netctl check` command will read `config.json`, then create a `DaemonSet` named `Floater` and some related resources, and then obtain all the `IP` information of `Floater`, and then enter in sequence Go to `Pod` and execute the `Ping` or `Curl` command. It should be noted that this operation is executed concurrently, and the degree of concurrency changes dynamically according to the `maxNum` parameter in `config.json`.
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

* During the execution of the `check` command, a progress bar will display the verification progress. After the command is executed, the check results will be printed and saved in the file `resume.json`.
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

* If you need to check the network connectivity between any two clusters in the Kosmos cluster federation, you can add the parameters `dstKubeConfig` and `dstImageRepository` to the configuration file `config.json`, so that you can check the network connectivity between the two clusters. .
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

* `netctl resume` command is used to check only the cluster nodes with problems during the first inspection during retesting. Because there are a large number of nodes in the online environment, a single inspection may take a long time to generate results, so we hope to retest only the nodes that were abnormal in the previous inspection. The `resume` command was developed for this reason. This command will read the `resume.json` file and recheck the previous abnormal node. We can repeatedly execute this command until there are no abnormal results and then perform a full check.
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

* `netctl clean` command is used to clean up all resources created by `NetDoctor`.
