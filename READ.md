# Installing k3s on Ubuntu 20.04 running on AWS EC2 Instances

Purpose of the this wiki is to give idea the knowledge of how to install and configure k3s on `Ubuntu 20.04` EC2 Instances

# K3S Installation Requirements

In this section we will give you general requirements for K3s Cluster

- K3s is very lightweight, but has some minimum requirements as outlined below.

- Whether you’re configuring a K3s cluster to run in a Docker or Kubernetes setup, each node running K3s should meet the following minimum requirements. You may need more resources to fit your needs.

## 1) Prerequisites

- Two nodes cannot have the same hostname.
- If all your nodes have the same hostname, use the `--with-node-id` option to append a random suffix for each node, or otherwise devise a unique name to pass with `--node-name` or `$K3S_NODE_NAME` for each node you add to the cluster.

## 2) Operating Systems

- K3s is expected to work on most modern Linux systems.
- Some OSs have specific requirements:
- If you are using Raspbian Buster, follow [these steps](https://rancher.com/docs/k3s/latest/en/advanced/#enabling-legacy-iptables-on-raspbian-buster) to switch to legacy iptables.
- If you are using Alpine Linux, follow [these steps](https://rancher.com/docs/k3s/latest/en/advanced/#additional-preparation-for-alpine-linux-setup) for additional setup.
- If you are using (Red Hat/CentOS) Enterprise Linux, follow [these steps](https://rancher.com/docs/k3s/latest/en/advanced/#additional-preparation-for-red-hat-centos-enterprise-linux) for additional setup.

## 3) Hardware

- Hardware requirements scale based on the size of your deployments. Minimum recommendations are outlined here.

- RAM: 512MB Minimum (we recommend at least 1GB)
- CPU: 1 Minimum

[This section](https://rancher.com/docs/k3s/latest/en/installation/installation-requirements/resource-profiling/) captures the results of tests to determine minimum resource requirements for the K3s agent, the K3s server with a workload, and the K3s server with one agent. It also contains analysis about what has the biggest impact on K3s server and agent utilization, and how the cluster datastore can be protected from interference from agents and workloads.

## 4) Disk

- K3s performance depends on the performance of the database. To ensure optimal speed, we recommend using an SSD when possible. Disk performance will vary on ARM devices utilizing an SD card or eMMC.

## 5) Networking

- The K3s server needs port `6443` to be accessible by all nodes.

- The nodes need to be able to reach other nodes over `UDP` port `8472` when Flannel VXLAN is used or over `UDP`ports `51820` and `51821` (when using IPv6) when Flannel Wireguard backend is used. The node should not listen on any other port. K3s uses reverse tunneling such that the nodes make outbound connections to the server and all kubelet traffic runs through that tunnel. However, if you do not use Flannel and provide your own custom CNI, then the ports needed by Flannel are not needed by K3s.

- If you wish to utilize the metrics server, you will need to open port 10250 on each node.

- If you plan on achieving high availability with embedded etcd, server nodes must be accessible to each other on ports `2379` and `2380.`

- Inbound Rules for K3s Server Nodes

|PROTOCOL|PORT|SOURCE|DESCRIPTION|
|---|---|---|---|
|TCP|6443|K3s agent nodes|Kubernetes API Server|
|UDP|8472|K3s server and agent nodes|Required only for Flannel VXLAN|
|UDP| 51820| K3s server and agent nodes| Required only for Flannel Wireguard backend|
|UDP| 51821| K3s server and agent nodes| Required only for Flannel Wireguard backend with IPv6|
|TCP| 10250| K3s server and agent nodes| Kubelet metrics|
TCP| 2379-2380| K3s server nodes| Required only for HA with embedded etcd|

- Typically all outbound traffic is allowed.

## 6) Large Clusters

- Hardware requirements are based on the size of your K3s cluster. For production and large clusters, we recommend using a high-availability setup with an external database. The following options are recommended for the external database in production:
- MySQL
- PostgreSQL
- etcd

### CPU and Memory

- The following are the minimum CPU and memory requirements for nodes in a high-availability K3s server:

|DEPLOYMENT SIZE|NODES|VCPUS|RAM|
|---|---|---|---|
|Small|Up to 10|2|4 GB|
|Medium|Up to 100|4|8 GB|
|Large| Up to 250| 8| 16 GB|
|X-Large| Up to 500| 16| 32 GB|
|XX-Large| 500+| 32| 64 GB|

### Disk

- The cluster performance depends on database performance. To ensure optimal speed, we recommend always using SSD disks to back your K3s cluster. On cloud providers, you will also want to use the minimum size that allows the maximum IOPS.

### Network

- You should consider increasing the subnet size for the cluster CIDR so that you don’t run out of IPs for the pods. You can do that by passing the `--cluster-cidr` option to K3s server upon starting.

### Database

- K3s supports different databases including MySQL, PostgreSQL, MariaDB, and etcd, the following is a sizing guide for the database resources you need to run large clusters:

|DEPLOYMENT SIZE|NODES|VCPUS|RAM|
|---|---|---|---|
|Small| Up to 10 |1 |2 GB|
|Medium| Up to 100| 2 |8 GB|
|Large| Up to 250 |4 |16 GB|
|X-Large| Up to 500| 8 |32 GB|
|XX-Large| 500+| 16 |64 GB|

## Part 1 - Setting Up k3s Environment on All Nodes

- First of all, we need to prepare 2 instances for k3s on 'ubuntu 20.04'.
- One of the instances will be `master`
- Other instance will be `agent` or known as `worker node`

### System Requirements for Project-X

- Requirements for project-X

- Virtual Machine Requirements:

|EC2 AMI Type|Architecture|Instance Type| VCPUS | RAM|
|---|---|---|---|---|
|Ubuntu| 64-bit (x86)| t3a.nano or t4g.nano| 2 core | 512 Mb|

- Database to store the cluster information:

- We will use RDS `MariaDB`

- we can use `free tier`

# Installation

- EC2 installation; Select `Ubuntu 20.04` SSD Volume Type and Instance type `t3a.nano` select the key pair. And make sure instance has access to SSH connection (Port 22).  We need 2 instance 1 Master 1 Worker so select the Number of instance = `2`.  

- RDS installation; Go to AWS RDS console and select `create database`

##### RDS - MariaDB

- Creation Method: standard create
- Engine option : MariaDB
- Version : MariaDB 10.6.8
- Templates : Free Tier
- DB instance identifier: k3s-cluster0db (named with related project)
- master username: k3s
- master password: k3s12345
- Instance configuration : leave it as is
- Storage: unenabled auto-scaling
- Additional configuration: unenable automated backup, unenable encryption , unenable enhanced monitoring, unenable auto minor upgrade
- create database

##### Connect the ec2 instances with SSH

- connect both of the instance `worker` and `master`
- always is better to update the machine before we use
- go to root account in both ec2 instances and apply followin code to both

```bash
apt-get install -y
```

- Hostname change of the nodes, so we can discern the roles of each nodes. For example, you can name the nodes (instances) like `k3s-master, k3s-worker`

```bash
hostnamectl set-hostname <node-name-master-or-worker>
bash
```

- We need to bootstrap the master node and then we have to attach the worker nodes to the master node. First thing is we need to connect to the RDS and create a database to store our cluster information in master node. To do this we need to install MysQL client.

- Install MysQL client on `Master Node`

```bash
apt install mysql-client -y
```

- Go to the AWS Console -> RDS -> MariaDB database on the `connectivity & security` section and copy `End point` of the database. example : k3s.cspat0oooh9v.us-east-1.rds.amazonaws.com

- Connect to the RDS on Master Node  example of the following bash -> mysql -h k3s.cspat0oooh9v.us-east-1.rds.amazonaws.com -u k3s -p and enter the password

```bash
mysql -h < paste here the endpoint > -u <masteruser of the rds> -p 
```

- It will ask you the password when you created at the beginning of the RDS installation enter it.

```bash

...
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 23
Server version: 5.5.5-10.6.8-MariaDB managed by 
https://aws.amazon.com/rds/

Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

```

- Now we connected the RDS let's see what are the available databases in there.

```bash
show databases;
```

- all default databases;

```bash
...
+--------------------+  
| Database           |  
+--------------------+  
| information_schema |  
| innodb             |  
| mysql              |  
| performance_schema |  
| sys                |  
+--------------------+  
5 rows in set (0.00 sec)
```

- Create a database with the name k3s to store our cluster information

```bash
CREATE DATABASE k3s;
```

```bash
...
mysql> CREATE DATABASE k3s;
Query OK, 1 row affected (0.00 sec)
```

- flush privileges

```bash
FLUSH PRIVILEGES;
```

- Quit from mysql

```bash
exit
```

- export

```bash
export K3S_DATASTORE_ENDPOINT='mysql://username:password@endpoint(hostname:3306)/database-name'
```

- example: export K3S_DATASTORE_ENDPOINT='mysql://k3s:k3s12345@tcp(k3s-cluster-db.cspat0oooh9v.us-east-1.rds.amazonaws.com:3306)/k3s'

-

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--write-kubeconfig=/home/ubuntu/.kube/config --write-kubeconfig-mode=644" sh - 
```

- check the status of k3s

```bash
systemctl status k3s
```

-

```bash
kubectl get nodes
```

- exit the root

```bash
exit
```

- list the files

```bash
ll
```

```bash
...
total 32
drwxr-xr-x 5 ubuntu ubuntu 4096 Aug  9 02:41 ./
drwxr-xr-x 3 root   root   4096 Aug  9 01:38 ../
-rw-r--r-- 1 ubuntu ubuntu  220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 ubuntu ubuntu 3771 Feb 25  2020 .bashrc
drwx------ 2 ubuntu ubuntu 4096 Aug  9 01:56 .cache/
drwxr-xr-x 2 root   root   4096 Aug  9 02:41 .kube/
-rw-r--r-- 1 ubuntu ubuntu  807 Feb 25  2020 .profile
drwx------ 2 ubuntu ubuntu 4096 Aug  9 01:38 .ssh/
-rw-r--r-- 1 ubuntu ubuntu    0 Aug  9 01:58 .sudo_as_admin_successful
```

-

```bash
```

-

```bash
```

-

```bash
```

-

```bash
```

-

```bash
```
