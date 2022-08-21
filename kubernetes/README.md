# Kubernetes

***Table of Contents***

* [Overview](#overview)
  * [Hardware](#hardware)
  * [Nodes](#nodes)
* [Tutorial](#tutorial)
  * [Setup Raspberry Pi](#setup-raspberry-pi)
  * [Basic Settings](#basic-settings)
  * [Ansible](#ansible)
  * [OS Settings](#os-settings)
  * [Central Logging](#central-logging)
    * [Logging server](#logging-server)
    * [Logging clients](#logging-clients)
  * [Install Kubernetes](#install-kubernetes)
    * [Control / Master Node](#control--master-node)
    * [Worker Nodes](#worker-nodes)
    * [Configure Kubectl On Client](#configure-kubectl-on-client)
  * [Network Settings](#network-settings)
  * [Helm & Arkade](#helm--arkade)
  * [Storage](#storage)
    * [Storage Disk Preparations](#storage-disk-preparations)
    * [Install Longhorn](#install-longhorn)
  * [Docker](#docker)
    * [Docker Installation](#docker-installation)
    * [Docker Registry](#docker-registry)
    * [Docker Registry TLS](#docker-registry-tls)
  * [Redis](#redis)
  * [Portainer](#portainer)
  * [PostgreSQL](#postgresql)
* [Useful Commands](#useful-commands)

## Overview

### Hardware

- 5x Raspberry Pi 4 8GB
- 5x Raspberry Pi Offical PowerSupply, 5.1V, 3A
- 5x SanDisk microSDXC 64GB
- 5x SanDisk Ultra Fit 128GB
- 1x Cluster case
- 1x Netgear network switch 8port

### Nodes

| Hostname             | IP            | Type                   |
| -------------------- | ------------- | ---------------------- |
| RPI-CLUSTER-01.local | 192.168.0.101 | control-plane / master |
| RPI-CLUSTER-02.local | 192.168.0.102 | worker                 |
| RPI-CLUSTER-03.local | 192.168.0.103 | worker                 |
| RPI-CLUSTER-04.local | 192.168.0.104 | worker                 |
| RPI-CLUSTER-05.local | 192.168.0.105 | worker                 |

### Exposed Services

| Name       | IP            |
| ---------- | ------------- |
| Traefik    | 192.168.0.200 |
| Longhorn   | 192.168.0.201 |
| Docker     | 192.168.0.202 |
| Redis      | 192.168.0.203 |
| Portainer  | 192.168.0.204 |
| PostgreSQL | 192.168.0.205 |

## Tutorial

### Setup Raspberry Pi

For each Raspberry Pi do the following:

1. Flash the SD-card
    - Open Raspberry Pi Imager
    - Pick Raspberry Pi OS Lite *(64-bit)*
    - Choose storage *(SD-card)*
    - Configure hostname, ssh, username, password *(for simplicity I recommend using the same username/password, maybe not that secure)*
    - Then click write
2. Insert the SD card into a Raspberry Pi and let it boot
3. Execute `ssh <username>@RPI-CLUSTER-0<N>.local` and perform a safe shutdown `sudo shutdown -P now`
4. Insert SD card into your computer and append `group_enable=cpuset cgroup_enable=memory cgroup_memory=1 ip=192.168.0.10<N>::192.168.0.1:255.255.255.0:RPI-CLUSTER-0<N>:eth0:off` to the end of the cmdline.txt on the boot partition. Make sure the ip-address and hostname are correct.
5. Insert the SD card into the current Raspberry Pi and let it boot up again
6. Execute `ssh <username>@RPI-CLUSTER-0<N>.local` again and verify the ip address with `ifconfig`

### Basic Settings

SSH into your control-plane / master* node `ssh <username>@RPI-CLUSTER-01.local` and verify that all nodes are up using `nmap`.

```bash
$ sudo -s
$ apt install nmap
$ nmap -sP 192.168.0.101-105
```

Edit `/etc/hosts` on the *control-plane / master* node **RPI-CLUSTER-01** and replace the contents with the following below:

```
# /etc/hosts

127.0.0.1 localhost

192.168.0.101 RPI-CLUSTER-01 RPI-CLUSTER-01.local
192.168.0.102 RPI-CLUSTER-02 RPI-CLUSTER-02.local
192.168.0.103 RPI-CLUSTER-03 RPI-CLUSTER-03.local
192.168.0.104 RPI-CLUSTER-04 RPI-CLUSTER-04.local
192.168.0.105 RPI-CLUSTER-05 RPI-CLUSTER-05.local
```

### Ansible

Finally we will make our lives easier with *Ansible*.

> Ansible® is an open sourceIT automation tool that automates provisioning, configuration management, application deployment, orchestration, and many other manual IT processes.

SSH into your control-plane / master* node `ssh <username>@RPI-CLUSTER-01.local` and install *Ansible*.

```bash
$ sudo apt install ansible
```

Create a file `/etc/ansible/hosts` and configure our hosts.

```
[control]
RPI-CLUSTER-01  ansible_connection=local

[workers]
RPI-CLUSTER-02  ansible_connection=ssh  ansible_user=<username>
RPI-CLUSTER-03  ansible_connection=ssh  ansible_user=<username>
RPI-CLUSTER-04  ansible_connection=ssh  ansible_user=<username>
RPI-CLUSTER-05  ansible_connection=ssh  ansible_user=<username>

[cluster:children]
control
workers
```

Now make it so that our default user will be able to log in to our worker nodes from **RPI-CLUSTER-01** without the password using an ssh key.

```bash
$ sudo -i
$ mkdir -p ~/.ssh
$ chmod 700 ~/.ssh

$ ssh-keygen -t rsa

$ ssh-copy-id -i ~/.ssh/id_rsa.pub <username>@RPI-CLUSTER-02.local
$ ssh-copy-id -i ~/.ssh/id_rsa.pub <username>@RPI-CLUSTER-03.local
$ ssh-copy-id -i ~/.ssh/id_rsa.pub <username>@RPI-CLUSTER-04.local
$ ssh-copy-id -i ~/.ssh/id_rsa.pub <username>@RPI-CLUSTER-05.local
```

Create a new *Ansible* config file in `~/.ansible.cfg` with the following content:

```
[defaults]
host_key_checking = False
deprecation_warnings = False
```

Verify that *Ansible* is working.

```
$ ansible cluster -m ping
> RPI-CLUSTER-01 | SUCCESS => {
     "ansible_facts": {
         "discovered_interpreter_python": "/usr/bin/python"
     },
     "changed": false,
     "ping": "pong"
 }
 RPI-CLUSTER-02 | SUCCESS => {
     "ansible_facts": {
         "discovered_interpreter_python": "/usr/bin/python"
     },
     "changed": false,
     "ping": "pong"
 }
 RPI-CLUSTER-04 | SUCCESS => {
     "ansible_facts": {
         "discovered_interpreter_python": "/usr/bin/python"
     },
     "changed": false,
     "ping": "pong"
 }
 RPI-CLUSTER-03 | SUCCESS => {
     "ansible_facts": {
         "discovered_interpreter_python": "/usr/bin/python"
     },
     "changed": false,
     "ping": "pong"
 }
 RPI-CLUSTER-05 | SUCCESS => {
     "ansible_facts": {
         "discovered_interpreter_python": "/usr/bin/python"
     },
     "changed": false,
     "ping": "pong"
 }
```

### OS Settings

Next we are going to update operating system packages to the latest versions. This can be done across all worker nodes with a single *Ansible* command.

```
$ ansible cluster -b -m apt -a "upgrade=yes update_cache=yes"
```

k3s / Kubernetes needs iptables so let's install it.

```bash
$ ansible cluster -b -m apt -a "name=iptables state=present"
```

Copy over our `/etc/hosts` file to all workers.

```bash
$ ansible workers -b -m copy -a "src=/etc/hosts dest=/etc/hosts owner=root group=root mode=0644"
```

Then reboot the whole cluster for good measure.

```bash
$ ansible cluster -b -m shell -a "reboot"
```

### Central Logging

#### Logging server

```bash
$ sudo -i
$ mkdir /var/log/central
```

Edit `/etc/rsyslog.conf` and uncomment these lines

```
# /etc/rsyslog.conf

# provides UDP syslog reception
module(load="imudp")
input(type="imudp" port="514")

# provides TCP syslog reception
module(load="imtcp")
input(type="imtcp" port="514")
```

Create a new config file `/etc/rsyslog.d/central.conf` with contents of:

```
$template RemoteLogs,"/var/log/central/%HOSTNAME%.log"
*.*  ?RemoteLogs
```

This will put all logs under /var/log/central/<hostname>.log

Create file `/etc/logrotate.d/central` with the contents below. This will tell logrotate to rotate the logs, so you don't end up with 100+MB text files.

```
# /etc/logrotate.d/central

/var/log/central/*.log
{
        rotate 4
        weekly
        missingok
        notifempty
        compress
        delaycompress
        sharedscripts
        postrotate
                invoke-rc.d rsyslog rotate >/dev/null 2>&1 || true
        endscript
}
```

Then restart rsyslog.

```bash
$ systemctl restart rsyslog
```

#### Logging clients

On each worker node put following line `*.* @@RPI-CLUSTER-01.local:514` at the start of `/etc/rsyslog.conf`.

It should look something like this:

```
# /etc/rsyslog.conf configuration file for rsyslog
#
# For more information install rsyslog-doc and see
# /usr/share/doc/rsyslog-doc/html/configuration/index.html
*.* @@RPI-CLUSTER-01.local:514

#################
#### MODULES ####
#################
...
...
...
```

Then restart rsyslog. *NOTE that you may need to fill in the *control-plane / master* node user's password.*

```bash
$ systemctl restart rsyslog
```

Now back at out logging server **RPI-CLUSTER-01**, check the contents of the folder `/var/log/central`.

```bash
$ ls /var/log/central
> RPI-CLUSTER-01.log  RPI-CLUSTER-02.log  RPI-CLUSTER-03.log  RPI-CLUSTER-04.log  RPI-CLUSTER-05.log
```

A nifty litte program called `lnav` lets you to watch your logs in real time, with filters and so on.

```bash
$ sudo apt install lnav
$ lnav /var/log/central/*.log
```

### Install Kubernetes

#### Control / Master Node
The version of Kubernetes we are about to install is called **K3s**. Use the following command to download and initialize K3s on our control/master node. Remember to replace the token / password with something you don't forget, you'll need it later on.

```bash
$ curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644 --disable servicelb --token '<SOME_RANDOM_PASSWORD>' --node-taint CriticalAddonsOnly=true:NoExecute --bind-address 192.168.0.101 --disable-cloud-controller --disable local-storage
```

If installation was successful, we can verify that our *control-plane / master* node is up and running.

```bash
$ kubectl get nodes
> NAME        STATUS   ROLES                  AGE   VERSION
  control01   Ready    control-plane,master   20s   v1.23.6+k3s1
```

#### Worker Nodes

We are going to execute the following *Ansible* command on each worker node in order to join them to out cluster.

```bash
$ ansible workers -b -m shell -a "curl -sfL https://get.k3s.io | K3S_URL=https://192.168.0.101:6443 K3S_TOKEN='<SOME_RANDOM_PASSWORD>' sh -"
```

Give it a couple of minutes for our workers to join the cluster. You can watch the progress by using the following command:

```
$ watch kubectl get nodes
> rpi-cluster-01   Ready    control-plane,master   1m0s   v1.24.3+k3s1
  rpi-cluster-05   Ready    <none>                 20s    v1.24.3+k3s1
  rpi-cluster-04   Ready    <none>                 20s    v1.24.3+k3s1
  rpi-cluster-03   Ready    <none>                 20s    v1.24.3+k3s1
  rpi-cluster-02   Ready    <none>                 20s    v1.24.3+k3s1
# to quit watch use Ctrl+C
```

We can now tag our cluster nodes and give them labels.

```bash
$ kubectl label nodes rpi-cluster-02 kubernetes.io/role=worker
$ kubectl label nodes rpi-cluster-03 kubernetes.io/role=worker
$ kubectl label nodes rpi-cluster-04 kubernetes.io/role=worker
$ kubectl label nodes rpi-cluster-05 kubernetes.io/role=worker
```

Another label/tag `node-type` to tell deployments to prefer nodes where node-type equals workers.

```bash
$ kubectl label nodes rpi-cluster-02 node-type=worker
$ kubectl label nodes rpi-cluster-03 node-type=worker
$ kubectl label nodes rpi-cluster-04 node-type=worker
$ kubectl label nodes rpi-cluster-05 node-type=worker
```

Lastly, add following into `/etc/environment` (this is so the Helm and other programs know where the Kubernetes config is.)

```bash
$ ansible cluster -b -m lineinfile -a "path='/etc/environment' line='KUBECONFIG=/etc/rancher/k3s/k3s.yaml'"
```

#### Configure Kubectl On Client

First install kubectl on your local computer you want to access the cluster. Now you have to add the the cluster to your local kubeconfig file. Therefore first ssh into the *control-plane / master* node and copy the content of following file.

```bash
$ sudo cat /etc/rancher/k3s/k3s.yaml
```

Copy the content into your local kubeconfig file that typically can be found in the home folder in `~/.kube/config`. Here it is crucial to replace the localhost IP *127.0.0.1* of the server with the actual IP address *(192.168.0.101)* of the *control-plane / master* node in your network.

Test that the configuration work by getting all nodes in the cluster.

```bash
$ kubectl get nodes
```

### Network Settings

K3s comes pre-configured with *Traefik* (pronounced traffic) a modern HTTP reverse proxy and load-balancer. We will instead use *MetalLB* load-balancer.

Begin by deploying MetalLB load balancer.

```bash
$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/namespace.yaml
$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/metallb.yaml
```

We need to create a secret key for the speakers (the MetalLB pods) to encrypt speaker communications:

```bash
$ kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

Apply the the *MetalLB* config:

```bash
$ kubectl apply -f kubernetes/namespaces/metallb-system/metallb-config.yaml
```

Check if everything deployed OK

```bash
$ kubectl get pods -n metallb-system
> NAME                          READY   STATUS    RESTARTS   AGE
  controller-7476b58756-p4bkv   1/1     Running   0          30s
  speaker-jxs86                 1/1     Running   0          30s
  speaker-82srk                 1/1     Running   0          30s
  speaker-5jssp                 1/1     Running   0          30s
  speaker-gj7jp                 1/1     Running   0          30s
```

You should have as many *speaker-xxxx* as you have worker nodes in the cluster, since they run one per node.

Now services that use LoadBalancer should have an external IP assigned to them.

```bash
$ kubectl get svc --all-namespaces
> NAMESPACE     NAME             TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
  default       kubernetes       ClusterIP      10.43.0.1      <none>          443/TCP                      6d1h
  kube-system   kube-dns         ClusterIP      10.43.0.10     <none>          53/UDP,53/TCP,9153/TCP       6d1h
  kube-system   metrics-server   ClusterIP      10.43.79.230   <none>          443/TCP                      6d1h
  kube-system   traefik          LoadBalancer   10.43.160.59   192.168.0.200   80:32420/TCP,443:32338/TCP   6d1h
```

### Helm & Arkade

Now install two package managers named *Helm* and *Arkade*.

Begin by SSH into the *control-plane / master* node and install *Helm*.

```bash
$ cd && mkdir helm && cd helm
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
> Downloading https://get.helm.sh/helm-v3.9.3-linux-arm64.tar.gz
  Verifying checksum... Done.
  Preparing to install helm into /usr/local/bin
  helm installed into /usr/local/bin/helm
```

Check if *Helm* was installed correctly.

```bash
$ helm version
> version.BuildInfo{Version:"v3.9.3", GitCommit:"414ff28d4029ae8c8b05d62aa06c7fe3dee2bc58", GitTreeState:"clean", GoVersion:"go1.17.13"}
```

Then install *Arkade*.

```bash
$ curl -SLsf https://dl.get-arkade.dev/ | sudo sh
```

Check if *Arkade* was installed correctly.

```bash
$ arkade version
> ...
  Version: 0.8.34
  Git Commit: b273819ddf3f78f5d23f5af204ed815638f97d56
```

### Storage

**Longhorn**

We are using Longhorn, a distributed block storage system for Kubernetes together with separate 128 GB USB flash-drives on each worker node to be a volume for storage. Longhorn is making this very simple for us. All you need to do is mount the disk under `/var/lib/longhorn`, which will be the default mount point under `/storage01`.

#### Storage Disk Preparations

**Software requirements**
Install following software, from master node run:

```bash
$ ansible cluster -b -m apt -a "name=nfs-common state=present"
$ ansible cluster -b -m apt -a "name=open-iscsi state=present"
$ ansible cluster -b -m apt -a "name=util-linux state=present"
```

**Identifying disks for storage**

We are going to use *Ansible* and add new variables with disk names that will be used for storage in `/etc/ansible/hosts`. 

```bash
$ ansible cluster -b -m shell -a "lsblk -f"
```

Edit `/etc/ansible/hosts` and add a new variable (I have chosen name var_disk) with the disk to wipe.

```
[control]
RPI-CLUSTER-01  ...  var_hostname=RPI-CLUSTER-01

[workers]
RPI-CLUSTER-02  ...  var_hostname=RPI-CLUSTER-02  var_disk=sdx
RPI-CLUSTER-03  ...  var_hostname=RPI-CLUSTER-03  var_disk=sdx
RPI-CLUSTER-04  ...  var_hostname=RPI-CLUSTER-04  var_disk=sdx
RPI-CLUSTER-05  ...  var_hostname=RPI-CLUSTER-05  var_disk=sdx

[cluster:children]
control
workers
```

**Wipe**

Now wipe all USB flash-drives with `wipefs -a`.

```bash
$ ansible cluster -b -m shell -a "wipefs -a /dev/{{ var_disk }}"
$ ansible cluster -b -m filesystem -a "fstype=ext4 dev=/dev/{{ var_disk }}"
```

**File system and mount**

In order to mount our storage disks we need their unique identifiers *(UUIDs)* in case their `/dev/...` labels changes.

```bash
$ ansible cluster -b -m shell -a "blkid -s UUID -o value /dev/{{ var_disk }}"
```

Then add these to `/etc/ansible/hosts`, with another custom variable `var_uuid`, for example:

```
[control]
RPI-CLUSTER-01  ...  var_uuid=7dd66e71-a48d-49b2-b68a-e5967d414915

[workers]
RPI-CLUSTER-02  ...  var_uuid=67e0f26d-5620-40a4-b1c3-39c1fb52afaf
RPI-CLUSTER-03  ...  var_uuid=ec6d0972-1db8-4a03-a492-705e08329033
RPI-CLUSTER-04  ...  var_uuid=59828619-56be-4dd1-ad49-a472595ff0be
RPI-CLUSTER-05  ...  var_uuid=7d667ada-77ec-483f-8acc-cd466405ed7a

[cluster:children]
control
workers
```

And finally using *Ansible* mount the disks to `/storage01`.

```bash
$ ansible cluster -b -m ansible.posix.mount -a "path=/storage01 src=UUID={{ var_uuid }} fstype=ext4 state=mounted"
```

#### Install Longhorn

On the *control / master* node:

```bash
$ helm repo add longhorn https://charts.longhorn.io
$ helm repo update
$ helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace --set defaultSettings.defaultDataPath="/storage01"
```

Verify that *Longhorn* is running. Everything under namespace longhorn-system should be *1/1 Running* or *2/2 Running*.

```bash
$ kubectl -n longhorn-system get pod
```

Look also at services. Longhorn-frontend is a management UI for storage.

```bash
$ kubectl -n longhorn-system get svc
```

**UI**

We will setup an external IP for the UI from the range we setup in metallb.

```bash
$ kubectl apply -f kubernetes/namespaces/longhorn/longhorn-service.yaml
```

And finally verify:

```bash
$ kubectl get svc --all-namespaces | grep longhorn-ingress-lb
```

**Make Longhorn the default StorageClass**

```bash
$ kubectl get storageclass
> NAME                 PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
  longhorn (default)   driver.longhorn.io   Delete          Immediate           true                   10m
```


### Docker

#### Docker Installation

Remove possibly current installation of docker.

```bash
$ sudo apt remove docker docker-engine docker.io containerd runc
```

Install prerequisites

```bash
$ sudo apt install ca-certificates curl gnupg lsb-release
```

Install GPG key

```bash
$ sudo mkdir -p /etc/apt/keyrings
$ curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

Install Repository

```bash
$ echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install Docker itself

```bash
$ sudo apt update
$ sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Add `/etc/docker/daemon.json` configuration for docker daemon.

```json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "insecure-registries": ["registry.cluster.local:5000"],
  "experimental": true,
  "log-driver": "json-file",
  "storage-driver": "overlay2",
  "log-opts": {
    "max-size": "100m"
  }
}
```

Enable at boot and start docker daemon

```bash
$ sudo systemctl enable docker
$ sudo systemctl start docker
```

#### Docker Registry

**Namespace**

Begin by creating a new docker registry namespace.

```bash
$ kubectl create namespace docker-registry
```

**Storage**

Apply our persistance volume claim.

```bash
$ kubectl apply -f kubernetes/namespaces/docker-registry/docker-registry-pvc.yaml
```

And verify it's existence:

```bash
$ kubectl get pvc -n docker-registry
> NAME                           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
  longhorn-docker-registry-pvc   Bound    pvc-bb91c624-7f63-45cb-9c02-cad45625a503   15Gi       RWO            longhorn       30s

$ kubectl get pv -n docker-registry
> NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                          STORAGECLASS   REASON   AGE
  pvc-bb91c624-7f63-45cb-9c02-cad45625a503   15Gi       RWO            Delete           Bound    docker-registry/longhorn-docker-registry-pvc   longhorn                58s
```

**Deployment**

Now we will create a simple deployment of docker registry and let it loose on our Kubernetes cluster.

Apply the deployment:

```bash
$ kubectl apply -f kubernetes/namespaces/docker-registry/docker-registry-deployment.yaml
> deployment.apps/registry created
```

And verify:

```bash
$ kubectl get deployments -n docker-registry
> NAME       READY   UP-TO-DATE   AVAILABLE   AGE
  registry   1/1     1            1           51s

$ kubectl get pods -n docker-registry
> NAME                        READY   STATUS    RESTARTS   AGE
  registry-6c956f54cf-nb5wb   1/1     Running   0          80s
```

**Service**

We use *MetalLb* as a LoadBalancer service for our Docker registry.

```bash
$ kubectl apply -f kubernetes/namespaces/docker-registry/docker-registry-service.yaml
> service/registry-service created
```

Give it a few seconds for the service to come up.

```bash
$ kubectl get svc -n docker-registry
> NAME               TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)          AGE
  registry-service   LoadBalancer   10.43.99.239   192.168.0.202   5000:31346/TCP   68s
```

**Local DNS**

Edit `/etc/hosts` and and the following line at the end `192.168.0.202 registry registry.cluster.local` on every node.

```bash
$ ansible cluster -b -m shell -a "echo \"192.168.0.202 registry registry.cluster.local\" >> /etc/hosts"
```

Now tell k3s about it. As root, create file `/etc/rancher/k3s/registries.yaml` with content:

```
mirrors:
  docker-registry:
    endpoint:
      - "http://registry.cluster.local:5000"
```

Send it to every control node of the cluster:

```
# make sure the directory exists
$ ansible workers -b -m file -a "path=/etc/rancher/k3s state=directory"

# copy the file
$ ansible workers -b -m copy -a "src=/etc/rancher/k3s/registries.yaml dest=/etc/rancher/k3s/registries.yaml"
```

**Test**


#### Docker Registry TLS

### Redis

**Namespace**

```bash
$ kubectl create namespace redis-system
```

**Persistent storage PVC**

```bash
$ kubectl apply -f kubernetes/namespaces/redis-system/redis-pvc.yaml
```

**Deployment**

```bash
$ kubectl apply -f kubernetes/namespaces/redis-system/redis-deployment.yaml
```

```bash
$ kubectl get pods -n redis-system
> NAME                            READY   STATUS    RESTARTS   AGE
  redis-server-7bc487ffdf-92rfq   1/1     Running   0          77s
```

**Service**

```bash
$ kubectl apply -f kubernetes/namespaces/redis-system/redis-service.yaml
```

```bash
$ kubectl get svc -n redis-system
> NAME           TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)          AGE
  redis-server   LoadBalancer   10.43.193.25   192.168.0.203   6379:30706/TCP   2m44s
```

**/etc/hosts**

```bash
$ echo '192.168.0.203 redis redis.cluster.local' >> /etc/hosts
```

```bash
$ ansible cluster -b -m copy -a "src=/etc/hosts dest=/etc/hosts"
```

**Test**

```bash
$ sudo apt install telnet -y
$ telnet redis.cluster.local 6379
```

### Portainer

**Install**

```bash
$ helm repo add portainer https://portainer.github.io/k8s/
$ helm repo update
$ helm install --create-namespace -n portainer-system portainer portainer/portainer
```

**Verify**

```bash
$ kubectl get pods -n portainer-system
> NAME                         READY   STATUS    RESTARTS   AGE
  portainer-544bd75f98-gxltc   1/1     Running   0          49s
```

```bash
$ kubectl get pvc -n portainer-system
> NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
  portainer   Bound    pvc-a4d3c6de-5263-4aa5-9940-c97802cf28ac   10Gi       RWO            longhorn       104s
```

**Expose Service**

```bash
$ kubectl create -f kubernetes/namespaces/portainer/portainer-service.yaml
> service/portainer created
```

**Restart Portainer**

```bash
$ kubectl get deployments -n portainer-system
> NAME        READY   UP-TO-DATE   AVAILABLE   AGE
  portainer   1/1     1            1           6m52s
```

```bash
$ kubectl scale --replicas=0 deployment portainer -n portainer-system
> deployment.apps/portainer scaled
```

```bash
$ kubectl get deployments -n portainer-system
> NAME        READY   UP-TO-DATE   AVAILABLE   AGE
  portainer   0/0     0            0           8m23s
```

```bash
$ kubectl scale --replicas=1 deployment portainer -n portainer-system
> deployment.apps/portainer scaled
```

```bash
$ kubectl get deployments -n portainer-system
> NAME        READY   UP-TO-DATE   AVAILABLE   AGE
  portainer   1/1     1            1           10m26s
```

### PostgreSQL

**Namespace**

```bash
$ kubectl create namespace postgres-system
> namespace/postgres-system created
```

```bash
$ kubectl create secret generic postgres-credentials \
  --from-literal=postgres-username=postgres \
  --from-literal=postgres-password='super-secret-password' \
  -n postgres-system
> secret/postgres-credentials created
```

```bash
$ kubectl create -f kubernetes/namespaces/postgres-system/postgres-storage.yaml
> persistentvolumeclaim/postgres-pvc created
```

```bash
$ kubectl create -f kubernetes/namespaces/postgres-system/postgres-deployment.yaml
> deployment.apps/postgres created
```

```bash
$ kubectl create -f kubernetes/namespaces/postgres-system/postgres-service.yaml
```

## Useful Commands

Using Ansible, change `cluster` from affecting whole cluster, to either only `workers` or `control`.

```bash
$ ansible cluster ...
$ ansible worker ...
$ ansible control ... # (quite useless)
```

**Ping**

```bash
$ ansible cluster -m ping
```

**Upgrade packages**

```
$ ansible cluster -b -m apt -a "upgrade=yes update_cache=yes"
```

**Shell command**

```
$ ansible cluster -b -m shell -a "echo 'Hello'"
```

**Reboot**

```bash
$ ansible cluster -b -m shell -a "reboot"
```

**Shutdown**

```bash
$ ansible cluster -b -m shell -a "halt"
```
