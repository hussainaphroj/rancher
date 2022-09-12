# Rancher HA setup
# Table of contents
1. [Introduction](#introduction)
2. [Architecture](#Architecture)
3. [Configure a fixed registration address](#registration)
4. [Launch the first server node, Install Kubernetes and Set up the RKE2 Server](#setuprke)
5. [Join additional server nodes](#additional)
6. [Install Rancher](#rancher)

## Introduction <a name="introduction"></a>
Rancher is a software stack to manage multiple Kubernetes clusters. It eases the operation and security challenges of managing the multiple Kubernetes clusters across any infrastructure. It also helps the teams with integrated tools for running containerized workloads.  
It is open source and developed by Rancher lab. The Rancher is acquired by SUSE in December 2020.
It can be used for managing multiple clusters, importing the existing clusters, and deploying the new cluster as well.
  
It provides infrastructure orchestration that is you can manage the clusters from the console and perform many clusters-related operations such as backup and restores of etcd, upgrading of the cluster, adding and removing the nodes, deploying the workload, enabling continuous delivery using, etc.  

It also provides the application catalog which is also known as `Marketplace`. The marketplace is nothing but Helm charts. These charts can be easily installed using the console. There are also server other repositories included, and you can add your own to deploy your charts/app to your cluster. The app deploying runs the helm command in the backend.  

Rancher can be deployed using the `RKE` and `RKE2`. The Rancher Kubernetes Engine(RKE) is a CNCF-certified Kubernetes distribution that runs entirely within Docker containers. The RKE2, also known as RKE Government, is a Kubernetes distribution that focuses on security and compliance for U.S. Federal Government entities.RKE2 is considered the next iteration of the Rancher Kubernetes Engine. RKE1 uses Docker for deploying and managing control plane components, and it also uses Docker as the container runtime for Kubernetes. By contrast, RKE2 launches control plane components as static pods that are managed by the kubelet. RKE2’s container runtime is containerd.

### Architecture <a name="Architecture"></a>
We will be going to set up the Rancher using RKE2 on Ubuntu 18.04 VMS. The architecture diagram of our current deployment is shown below:
![Alt text](Rancher-ha.png "Architecture diagram")   

An HA RKE2 cluster consists of:

* A fixed registration address that is placed in front of server nodes to allow other nodes to register with the cluster
* An odd number (three recommended) of server nodes that will run etcd, the Kubernetes API, and other control plane services

The curent system details of Racher set up as:

| Hostname                | Ip address      | 
| ------------------------| --------------- |
| rancher01               | 192.168.56.101  |
| rancher02               |  192.168.56.102 |
| rancher03               | 192.168.56.103  |
| rancher-vip.example.com | 192.168.56.20   |
| rancher.example.com     | 5.195.2.3       |


 Setting up an HA cluster requires the following steps:

* Configure a fixed registration address
* Launch the first server node, Install Kubernetes and Set up the RKE2 Server
* Join additional server nodes

### Configure a fixed registration address <a name="registration"></a>
Server nodes beyond the first one and all agent nodes need a URL to register against. This endpoint can be set up using any number approaches, such as:  
* A layer 4 (TCP) load balancer
* Round-robin DNS
* Virtual or elastic IP addresses  
This endpoint can also be used for accessing the Kubernetes API. So you can, for example, modify your kubeconfig file to point to it instead of a specific node.  

Note that the rke2 server process listens on port 9345 for new nodes to register. The Kubernetes API is served on port 6443, as normal.  

We will be using Virtual IP address such as `rancher-vip.example.com`  

### Launch the first server node, Install Kubernetes and Set up the RKE2 Server <a name="setuprke"></a>
The first server node establishes the secret token that other server or agent nodes will register with when connecting to the cluster.  
Steps of installing and configuring the server node are:
* Add the nodes details in `/etc/hosts` files:
```
192.168.56.101	rancher01
192.168.56.102	rancher02
192.168.56.103	rancher03
192.168.56.20	rancher-vip.example.com
```
* Create a direcroty and creat the RKE2 config file manually.

To avoid certificate errors with the fixed registration address, you should launch the server with the tls-san parameter set. This option adds an additional hostname or IP as a Subject Alternative Name in the server’s TLS cert, and it can be specified as a list if you would like to access via both the IP and the hostname.  

First, you must create the directory where the RKE2 config file is going to be placed:
```
mkdir -p /etc/rancher/rke2/
```
Next, create the RKE2 config file at /etc/rancher/rke2/config.yaml using the following example:
```
tls-san:
  - rancher-vip.example.com
  - 192.168.56.101
  - rancher01
  - 192.168.56.102
  - rancher02
  - 192.168.56.103
  - rancher03
  ```
* Run the installer and enable, then start, rke2
```
curl -sfL https://get.rke2.io | sh -
systemctl enable rke2-server.service
systemctl start rke2-server.service
```
`rke2-server.service` will take some time to come up. you can observe its logs using `journalctl -u rke2-server -f` command  
`Note:` Additional utilities will be installed at `/var/lib/rancher/rke2/bin/`. They include: `kubectl`, `crictl`, and `ctr`.  

Two cleanup scripts will be installed to the path at `/usr/local/bin/rke2`. They are: `rke2-killall.sh` and `rke2-uninstall.sh`. A kubeconfig file will be written to `/etc/rancher/rke2/rke2.yaml`.A token that can be used to register other server or agent nodes will be created at `/var/lib/rancher/rke2/server/node-token`

* Copy the `kubectl` and configure `kubecofig`
```
cp /var/lib/rancher/rke2/bin/kubectl /usr/local/bin/
chmod +x /usr/local/bin/kubectl
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
kubectl get nodes -o wide
This will give output as:
NAME               STATUS   ROLES                       AGE     VERSION          INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
rancher01   Ready    control-plane,etcd,master   3m31s   v1.23.9+rke2r1   192.168.56.101   <none>        Ubuntu 18.04.4 LTS   5.4.0-100-generic   containerd://1.5.13-k3s1
```
You can also use the kubectl from rancher direcory itself
```
/var/lib/rancher/rke2/bin/kubectl \
        --kubeconfig /etc/rancher/rke2/rke2.yaml get pods --all-namespaces
```
### Join additional server nodes <a name="additional"></a>
Additional server nodes are launched much like the first, except that you must specify the server and token parameters so that they can successfully connect to the initial server node.

Steps to configure 2nd and 3rd nodes are:
* Add the nodes details in `/etc/hosts` files:
```
192.168.56.101	rancher01
192.168.56.102	rancher02
192.168.56.103	rancher03
192.168.56.20	rancher-vip.example.com
```
* Create a direcroty and creat the RKE2 config file manually.

First, you must create the directory where the RKE2 config file is going to be placed:
```
mkdir -p /etc/rancher/rke2/
```
Next, create the RKE2 config file at /etc/rancher/rke2/config.yaml using the following example. Please note that token can be found in `/var/lib/rancher/rke2/server/node-token` file on the server node.
```
token: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
server: https://rancher-vip.example.com:9345
tls-san:
  - rancher-vip.example.com
  - 192.168.56.101
  - rancher01
  - 192.168.56.102
  - rancher02
  - 192.168.56.103
  - rancher03
  ```
* Run the installer and enable, then start, rke2
```
curl -sfL https://get.rke2.io | sh -
systemctl enable rke2-server.service
systemctl start rke2-server.service
```
`rke2-server.service` will take some time to come up. you can observe its logs using `journalctl -u rke2-server -f` command

* Confirm cluster is functional¶
Once you've launched the rke2 server process on all server nodes, ensure that the cluster has come up properly with
```
/var/lib/rancher/rke2/bin/kubectl \
        --kubeconfig /etc/rancher/rke2/rke2.yaml get nodes
```
This *conclude* the `Kubernetes` and `RKE2` installation , Next we will Setup the `Rancher` on server or 1st node

# Install Rancher <a name="rancher"></a>
We will install the `Rancher` using `Helm` on the first server node only.  
Steps for Installing Rancher:
* Install Helm
```
wget https://get.helm.sh/helm-v3.9.4-linux-amd64.tar.gz
 tar -xvzf helm-v3.9.4-linux-amd64.tar.gz
  linux-amd64/helm /usr/local/bin/helm
  ```
  * Add rancher Helm repo
  ```
  helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
  helm repo update
  ```

  * Create `cattle-system` namespace
  ```
  kubectl create namespace cattle-system
  ```
  * Install Rancher
  ```
  helm install rancher rancher-stable/rancher --namespace cattle-system --set hostname=rancher.example.com --set bootstrapPassword='Password@123' --set tls=external
  ```
  * Verify the installation
  ```
  kubectl get all -n cattle-system
NAME                           READY   STATUS    RESTARTS   AGE
pod/rancher-86cccxxxxx-lrh7z   1/1     Running   0          93s
pod/rancher-86cccxxxxx-nqt85   1/1     Running   0          93s
pod/rancher-86cccxxxxx-rjggf   1/1     Running   0          93s
```
  *rancher.example.com* is  a load balancer  
  *bootstrapPassword* is used to log in Rancher UI  
  *NOTE:* Rancher may take several minutes to fully initialize. Please standby while Certificates are being issued, Containers are started and the Ingress rule comes up. If you don't have load balancer then you can manually point `rancher.example.com` to one of the server node for exampler `192.168.56.101` and browse the `https://rancher.example.com` to access the rancher using `admin` username and `Password@123` as password


  ![Alt text](rancher1.png "rancher log in page") 

  ![Alt text](rancher2.png "rancher Home page")


