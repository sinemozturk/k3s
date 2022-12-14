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
