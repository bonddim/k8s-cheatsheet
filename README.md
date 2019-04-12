# Links
* Presentation: https://docs.google.com/presentation/d/10RURTFacQjXIpQP6FWoT8B5cohPUQsfrIjnD4VBKMFM/edit?usp=sharing
* cheatsheet: https://github.com/olebel/k8s-cheatsheet 

## 1. Prerequisites (master and worker)
#### 1.1 Install docker
```
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce docker-ce-cli containerd.io
systemctl enable docker
systemctl start docker
```
#### 1.2 Disable swap
```
swapoff -a
```
```
# edit /etc/fstab, remove swap line
```
#### 1.3 Disable selinux
```
setenforce Permissive
```
```
# edit /etc/selinux/config
SELINUX=permissive
```
#### 1.4 Tune sysctl
```
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```
## 2. Install kubeadm (master and worker)
#### 2.1 Configure repositories
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF
```
#### 2.2 Install packages
```
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```
#### 2.3 Enable/start kubelet service
```
systemctl enable kubelet
systemctl start kubelet
```
#### 2.3.1 Check kubelet service logs
```
journalctl -u kubelet -f
```

## 3. Configure master node
#### 3.1 Init master node
###### Note: save worker join command appear in output
```
kubeadm init
```
#### 3.2 Configure local kubectl access
```
export KUBECONFIG=/etc/kubernetes/admin.conf
```
#### 3.3 Check pods, containers, kubelet running
```
docker ps
kubectl get pods --all-namespaces
systemctl status kubelet
```

#### 3.4 Install network plugin: Weave
```
# get deployment yaml
curl --location "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')" -o weave.yaml
# look inside
less weave.yaml
# apply
kubectl apply -f weave.yaml
```
#### 3.5 Generate workers join command if you forgot it from 3.1
```
kubeadm token create --print-join-command
```

## 4. Configure worker
#### 4.1
```
kubeadm join 10.0.2.15:6443 ...
```

# Troubleshooting ( differs on AWS/Virtualbox)
#### 1.
```
# check IP addresses on both machines
ifconfig
```
#### Problem: Kubernetes services uses inteface with default gateway configured.
```
# reset cluster configuration (master, worker)
kubeadm reset
```
#### 1.1
```
# recreate cluster advertising listening on private interface
kubeadm init --apiserver-advertise-address=172.28.28.10
# configure
kubeadm apply -f weave.yaml
# check pods, nodes
kubectl get svc
kubectl get nodes -o wide
kubectl get pods -o wide
```
```
# Join worker node to cluster
kubeadm token create --print-join-command
kubeadm join ...
```
#### 2.
```
# check pods
kubectl get pods -n kube-system
kubectl describe pods -n kube-system weave-net
# check containers on worker node
docker ps
docker logs ?
```
#### Problem: weave-net container can't connect to cluster service IP. No routing.
#### 2.1
```
# Add required route ( each worker )
route add <cluster_ip> gw <master_ip>
# route add 10.96.0.1 gw 172.28.28.10
# from worker test
ping 10.96.0.1
telnet 10.96.0.1 443
```
```
# monitor nodes, pods status
kubectl get nodes
kubectl get pods -n kube-system
```
#### 3. Deploy hello-minicube
```
kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node
```
##### never ends
```
kubectl get pods
kubectl describe pods
```
```
# delete deployment
kubectl get deployments
kubectl delete deployment hello-node
```
#### 3.1
```
kubeadm reset
kubeadm init --apiserver-advertise-address=172.28.28.10
kubeadm apply -f weave.yaml
```
```
# Join worker node to cluster
kubeadm token create --print-join-command
kubeadm join ...
```
```
kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node
kubectl get pods -o wide
```
```
# check app from worker and master node
curl <pod_ip>:8080
```
