# Installation d'un Cluster Kubernetes via kubeadm
Follow this documentation to set up a Kubernetes cluster on __Ubuntu 20.04 LTS__.

This documentation guides you in setting up a cluster with one master node and one worker node.

## Assumptions
|Role|FQDN|IP|OS|RAM|CPU|
|----|----|----|----|----|----|
|Master|kmaster.example.com|172.16.16.100|Ubuntu 18.04|2G|2|
|Worker|kworker.example.com|172.16.16.101|Ubuntu 18.04|2G|1|

## Au niveau du Master et du Worker
Perform all the commands as root user unless otherwise specified
##### Désactiver le firewall
```
ufw disable
```
##### Désactiver le Swap
```
swapoff -a; sed -i '/swap/d' /etc/fstab
```
##### Mettre á jour les paramétres sysctl pour kubernetes
```
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```
##### Installer Docker
```
apt install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
apt update
apt install -y docker-ce containerd.io
```
### Installation kubernetes
##### Ajout du repo apt
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
```
##### Installer les conposants de kubernetes
```
apt update && apt install -y kubeadm kubelet kubectl
```
## Au niveau du Master
##### Initialiser le Cluster
Update the below command with the ip address of kmaster
```
kubeadm init --apiserver-advertise-address=172.16.16.100 --pod-network-cidr=192.168.0.0/16
```
##### Deployer le réseau Calico
```
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml
```

##### Afficher la commande de jointure
```
kubeadm token create --print-join-command
```
## Au niveau du Worker
##### Rejoindre le Cluster
Use the output from __kubeadm token create__ command in previous step from the master server and run here.

## Vérification du Cluster
##### Afficher l'état des noeuds du Cluster
```
kubectl get nodes
```
##### Afficher des informations sur le Cluster
```
kubectl cluster-info
```
