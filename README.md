# Create VM
##### Install Virtualbox:  
`yum install VirtualBox-6.1`

##### Configure host-only network:  
_`$ vboxmanage hostonlyif create`_
    ```
    0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
    Interface 'vboxnet0' was successfully created
    ```

##### disable DHCP; set IPv4 address to 192.168.1.1:  
_`$ vboxmanage dhcpserver modify --interface=vboxnet0 --disable`_
_`$ vboxmanage hostonlyif ipconfig vboxnet0 --ip 192.168.1.1 --netmask 255.255.255.0`_

##### to see hostly network config  
_`$ vboxmanage list hostonlyifs`_

##### Create base VM:  
a) Get minimal installation ISO from https://yum.oracle.com/oracle-linux-isos.html  
_`$ cd ~`_  
_`$ mkdir -p k8s/vms/image`_  
_`$ mkdir -p k8s/vms/kmaster`_  
_`$ cd k8s/vms/image`_  
_`$ wget https://yum.oracle.com/ISOS/OracleLinux/OL7/u9/x86_64/x86_64-boot-uek.iso`_  

b) Create master VM:  
_`$ vboxmanage createvm --name "kmaster" --ostype Oracle_64 --basefolder vms/kmaster --register`_  
_`$ vboxmanage modifyvm "kmaster" --memory 4096 --cpus 2 --cpuhotplug on --acpi on --nic1 nat --natpf1 "ssh,tcp,,64123,,22" --nic2 hostonly --hostonlyadapter2 vboxnet0 --vrde on --vrdeport 33391`_  
_`$ vboxmanage createmedium --filename vms/kmaster/kmaster.vdi --size 50000`_  
_`$ vboxmanage storagectl "kmaster" --name "IDE Controller" --add ide`_  
_`$ vboxmanage storageattach "kmaster" --storagectl "IDE Controller" --port 0 --device 0 --type hdd --medium vms/kmaster/kmaster.vdi`_  
_`$ vboxmanage storageattach "kmaster" --storagectl "IDE Controller" --port 1 --device 0 --type dvddrive --medium image/x86_64-boot-uek.iso`_  

c) Start vm initially (from GUI virtualBox app)  

d) follow Oracle Linux installation:  
Note: OL Repo URL: http://yum.oracle.com/repo/OracleLinux/OL7/latest/x86_64

e) after completing installation, shut it off, start in headless mode and ssh to vm from terminal:  
_`$ vboxheadless --startvm kmaster &`_  
_`$ ssh localhost -p 64123`_  

# Install and configure k8s kubernetes cluster

a) configure kmaster vm (from ssh console):  
_`$ ssh localhost -p 64123`_  

b) update packages:  
_`$ sudo yum -y update`_  

c) disable SE Linux:  
_`$ sudo setenforce 0`_  
_`$ sudo sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux`_  

d) switch to root:  
_`$ sudo su -`_  

e) set hostname:  
_`# hostnamectl set-hostname 'k8s-master'`_  

f) configure firewall:  
_`# firewall-cmd --permanent --add-port=6443/tcp`_  
_`# firewall-cmd --permanent --add-port=2379-2380/tcp`_  
_`# firewall-cmd --permanent --add-port=10250/tcp`_  
_`# firewall-cmd --permanent --add-port=10251/tcp`_  
_`# firewall-cmd --permanent --add-port=10252/tcp`_  
_`# firewall-cmd --permanent --add-port=10255/tcp`_  
_`# firewall-cmd --reload`_  
_`# modprobe br_netfilter`_  
_`# echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables`_  

g) add hostnames:  
_`# vi /etc/hosts`_     
   ```
    192.168.1.30 k8s-master  
    192.168.1.40 worker-node1  
    192.168.1.50 worker-node2  
   ```

h) disable swap:  
_`# swapoff -a`_  
_`# vi /etc/fstab -> comment out swap partition`_  

i) add kubernetes repo:    
   ```
    # cat <<EOF > /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
            https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
    EOF
   ```

j) install kubeadm:  
_`# yum install kubeadm -y`_  

k) Enable additional oracle repos:  
_`# cd /etc/yum.repos.d/`_  
_`# vi oracle-linux-ol7.repo`_  
```
[ol7_latest]
name=Oracle Linux $releasever Latest ($basearch)
baseurl=http://yum.oracle.com/repo/OracleLinux/OL7/latest/$basearch/
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle
gpgcheck=1
enabled=1

[ol7_addons]
name=Oracle Linux $releasever Add ons ($basearch)
baseurl=http://yum.oracle.com/repo/OracleLinux/OL7/addons/$basearch/
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle
gpgcheck=1
enabled=1
```

l) install docker:  
_`# yum install docker`_

m) set the cgroupdriver to systemd:  
_`# vi /usr/lib/systemd/system/docker.service`_  
  add to `ExecStart=` line:    
`--exec-opt native.cgroupdriver=systemd`  
_`# systemctl daemon-reload`_  

n) start & enable docker and kubelet:  
_`# systemctl start docker`_  
_`# systemctl enable docker`_  
_`# systemctl start kubelet`_  
_`# systemctl enable kubelet`_  

##### clone master vm (shut it down before) to 2 node vms: `worker-node-1 & worker-node2`
a) clone from VirtualBox UI.  

b) change NAT forwarding ports to `64124 & 64125` accordingly  

c) start them:  
_`# vboxheadless --startvm worker-node1 &`_  
_`# vboxheadless --startvm worker-node2 &`_  

d) ssh to each node and change hostname:  
_`# ssh localhost -p 64124`_  
_`# sudo hostnamectl set-hostname 'worker-node1'`_  
_`# ssh localhost -p 64125`_  
_`# sudo hostnamectl set-hostname 'worker-node2'`_  

##### configure k8s on master:  
a) run kubeadm init as regular user:  
 - verification:  
   _`$ sudo kubeadm init --pod-network-cidr=192.168.1.0/16 --apiserver-advertise-address 192.168.1.30 --apiserver-cert-extra-sans 192.168.1.30 --dry-run`_  
 - actual configuration:  
   _`$ sudo kubeadm init --pod-network-cidr=192.168.1.0/16 --apiserver-advertise-address 192.168.1.30 --apiserver-cert-extra-sans 192.168.1.30`_  
```
W1112 11:21:37.114613   12059 configset.go:348] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.19.4
[preflight] Running pre-flight checks
	[WARNING Firewalld]: firewalld is active, please ensure ports [6443 10250] are open or your cluster may not function correctly
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.1.30]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s-master localhost] and IPs [192.168.1.30 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s-master localhost] and IPs [192.168.1.30 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 15.502288 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.19" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node k8s-master as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node k8s-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: heofif.w9ro5ccg7z050r38
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.30:6443 --token heofif.w9ro5ccg7z050r38 \
    --discovery-token-ca-cert-hash sha256:eefc797d8d72ec10e7dcd73d0f492e40ef1fce7b2d51934c861154df72b8e6e9
```

b) to make kubectl to run as non-root user:  
_`$ mkdir -p $HOME/.kube`_  
_`$ cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`_  
_`$ chown $(id -u):$(id -g) $HOME/.kube/config`_  

c) verify initialization:  
_`$ kubectl get nodes`_    
```
NAME         STATUS     ROLES    AGE   VERSION
k8s-master   NotReady   master   31m   v1.19.3
```
_`$ kubectl get pods --all-namespaces`_  
```
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   coredns-f9fd979d6-27rsv              0/1     Pending   0          92s
kube-system   coredns-f9fd979d6-nlczs              0/1     Pending   0          92s
kube-system   etcd-k8s-master                      1/1     Running   0          101s
kube-system   kube-apiserver-k8s-master            1/1     Running   0          101s
kube-system   kube-controller-manager-k8s-master   1/1     Running   0          101s
kube-system   kube-proxy-2txl9                     1/1     Running   0          92s
kube-system   kube-scheduler-k8s-master            1/1     Running   0          101s
```

d) deploy a pod network to the cluster (choose calico: https://docs.projectcalico.org/getting-started/kubernetes/quickstart):  
_`$ kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml`_  
```
namespace/tigera-operator created
podsecuritypolicy.policy/tigera-operator created
serviceaccount/tigera-operator created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/installations.operator.tigera.io created
customresourcedefinition.apiextensions.k8s.io/tigerastatuses.operator.tigera.io created
clusterrole.rbac.authorization.k8s.io/tigera-operator created
clusterrolebinding.rbac.authorization.k8s.io/tigera-operator created
deployment.apps/tigera-operator created
```

Because pods IP range is different than calico default one, we are not running this command:  
_`$ kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml`_    
but:  
_`$ wget https://docs.projectcalico.org/manifests/custom-resources.yaml`_  
_`$ vi custom-resources.yaml # change IP range to 192.168.1.0/16`_  
_`$ kubectl create -f custom-resources.yaml`_  
```
installation.operator.tigera.io/default created
```

Note:  
To change custom resource CIDR after pod network deployment:  
 - download yaml file  
 - edit it  
 - run  
_`$ kubectl delete installation.operator.tigera.io/default`_  
_`$ kubectl create -f custom-resources.yaml`_  
```
installation.operator.tigera.io/default created
```

e) verify k8s config:  
_`$ kubectl get pods --all-namespaces`_  
```
NAMESPACE         NAME                                       READY   STATUS    RESTARTS   AGE
calico-system     calico-kube-controllers-85ff5cb957-gljmz   1/1     Running   0          25s
calico-system     calico-node-jc46m                          1/1     Running   0          25s
calico-system     calico-typha-6786f8dbf6-bgsz5              1/1     Running   0          25s
kube-system       coredns-f9fd979d6-5zpds                    1/1     Running   0          13m
kube-system       coredns-f9fd979d6-s2mp2                    1/1     Running   0          13m
kube-system       etcd-k8s-master                            1/1     Running   0          13m
kube-system       kube-apiserver-k8s-master                  1/1     Running   0          13m
kube-system       kube-controller-manager-k8s-master         1/1     Running   0          13m
kube-system       kube-proxy-kjlhm                           1/1     Running   0          13m
kube-system       kube-scheduler-k8s-master                  1/1     Running   0          13m
tigera-operator   tigera-operator-5f668549f4-gs9l7           1/1     Running   0          54s
```

_`$ kubectl get pods -n calico-system`_  
```
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-85ff5cb957-gljmz   1/1     Running   0          44s
calico-node-jc46m                          1/1     Running   0          44s
calico-typha-6786f8dbf6-bgsz5              1/1     Running   0          44s
```

##### Join nodes
a) ssh to worker node:  
_`$ ssh localhost -p 64124`_  
_`$ ssh localhost -p 64125`_  
_`$ sudo su -`_  
_`$ kubeadm join 192.168.1.30:6443 --token heofif.w9ro5ccg7z050r38 --discovery-token-ca-cert-hash sha256:eefc797d8d72ec10e7dcd73d0f492e40ef1fce7b2d51934c861154df72b8e6e9`_  
```
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

b) verify they are visible on master node:  
_`$ ssh localhost -p 64123`_  
_`$ kubectl get nodes`_  
```
NAME           STATUS   ROLES    AGE    VERSION
k8s-master     Ready    master   17m    v1.19.3
worker-node1   Ready    <none>   2m8s   v1.19.3
worker-node2   Ready    <none>   92s    v1.19.3
```

# ToDo
Create terraform package for it.  

# Appendix
## some troubleshooting steps  
##### add ip to cert
https://blog.scottlowe.org/2019/07/30/adding-a-name-to-kubernetes-api-server-certificate/  
 - get conf file:  
_`$ kubectl -n kube-system get configmap kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' > kubeadm.yaml`_  
 - add IP (this is additional IP to existing ones):  
_`$ vi kubeadm.yaml`_  
```
apiServer:
  certSANs:
  - "192.168.1.30"
  extraArgs:
    advertise-address: 192.168.1.30
.....
```
 - delete old certs and create new ones:  
_`$ sudo mv /etc/kubernetes/pki/apiserver.{crt,key} ~`_  
_`$ sudo kubeadm init phase certs apiserver --config kubeadm.yaml`_  
```
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.0.2.15 192.168.1.30]
```
- get API server container id and kill it:  
_`$ sudo docker ps | grep kube-apiserver | grep -v pause | sed s/\ .*//`_  
_`$ sudo docker kill <containerID>`_  
_`$ sudo docker kill 6b5a74605555`_  

##### Disable firewall
_`$ sudo systemctl disable firewalld`_  
_`$ sudo systemctl stop firewalld`_  

##### Some VirtualBox cli commands
 - show running vms:  
_`$ vboxmanage list runningvms`_  
```
"kmaster" {3ebd8a3b-61e1-4e1a-ab6a-74a387432e18}
"worker-node1" {246ccb6e-e8b5-44af-86bb-63869f70e81b}
"worker-node2" {0fa461e9-db77-42e4-b08a-831deb5a9677}
```
 - list OS Types:  
_`$ vboxmanage list ostypes`_  

 - Disconnect ISO image from CD drive:    
_`$ vboxmanage storageattach "kmaster" --storagectl "IDE Controller" --port 1 --device 0 --type dvddrive --medium emptydrive`_  

 - show VM info:  
_`$ vboxmanage showvminfo kmaster`_  

 - remove ISO image from VM:    
_`$ vboxmanage closemedium dvd 098374db-0b7d-4261-a862-a053c4f784de <uuid-from-sshowvminfo>`_  

 - start headless vm:    
_`$ vboxheadless --startvm kmaster &`_  
_`$ vboxheadless --startvm olk &`_  
 
 - stop vm:  
_`$ vboxmanage controlvm kmaster poweroff`_  

 - show all vms:  
_`$ vboxmanage list vms`_  

 - other kubernetes info:    
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/  

 - clean cluster:  
_`$ sudo kubeadm reset`_  
_`$ rm -f $HOME/.kube/config`_  

