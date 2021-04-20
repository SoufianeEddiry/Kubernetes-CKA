Dans cette mise en pratique, vous allez mettre en place un cluster Kubernetes à l'aide de *kubeadm*.

# Quelques prérequis

##  Hardware Requirements

Pour la mise en place d'un cluster avec kubeadm, il est nécessaire d'avoir une ou plusieurs machines avec les spécifications suivantes (documentation officielle de Kubernetes):

- Système d'exploitation
  * Ubuntu 16.04+
  * Debian 9+
  * CentOS 7
  * Red Hat Enterprise Linux (RHEL) 7
  * Fedora 25+
  * HypriotOS v1.0.1+
  * Container Linux (tested with 1800.6.0)
- 2 GB RAM minimum par machine
- 2 CPUs minimum par machine

Dans cet exemple nous considérons 3 machines virtuelles basées sur Ubuntu 18.04, sur lesquelles nous avons un accès ssh. Nous ferons référence à ces machines en tant que *node1*, *node2* et *node3*.

Afin de suivre cet exercice, vous pouvez par exemple créer des machines virtuelles avec l'une des méthodes suivantes:

- en local par exemple en utilisant [Multipass](https://multipass.run)

Une fois multipass installé, vous n'aurez plus qu'à lancer la commande suivante pour créer vos 3 machines virtuelles:

```
$ for i in $(seq 1 3); do
  multipass launch -n node$i -c 2 -m 2G -d 10G
done
```

- en local en utilisant [Vagrant](https://vagrantup.com) et [VirtualBox](https://virtualbox.org) ou une autre solution de virtualisation

- sur l'infrastructure d'un cloud provider

> Attention:
Si vous souhaitez créer vos machines virtuelles sur l'infrastructure d'un cloud provider (Google Compute Engine, Amazon AWS, Packet, Rackspace, DigitalOcean, Civo, Scaleway, OVH, ...) l'instantiation de VMs est payante (peu cher pour un test de quelques minutes cependant).

## Kubectl

Assurez-vous d'avoir installé *kubectl* sur votre machine locale (vous pouvez vous reporter à un exercice précédent pour cela). Ce binaire permet de communiquer avec un cluster Kubernetes depuis la ligne de commande.

# Configuration

L'étape de configuration consiste à installer les logiciels nécessaires sur les 3 machines que vous avez créées à l'étape précédente.

Il y a différentes façon de réaliser cette configuration:
- en se connectant manuellement en ssh sur chaque machine
- en utilisant un utilitaire de configuration, comme *Ansible*, *Chef*, *Puppet*

Nous lancerons ici des commandes via ssh mais n'hésitez pas à passer par une autre méthode si vous le souhaitez.

## Installation des prérequis

Sur chaque machine, nous allons installer les éléments suivants:
- un runtime de container (nous utiliserons Docker)
- le binaire *kubeadm* pour la création du cluster
- le binaire *kubelet* pour la supervision des containers

Connectez-vous en ssh sur chacune de vos machines virtuelles et lancez les commandes suivantes:

Note:
- assurez-vous au préalable que votre utilisateur a les droits *sudo*
- par défaut, la dernière version de Kubernetes va être installée mais il est possible d'installer une version précédente. Il faudra pour cela remplacer la dernière ligne de la commande : ```sudo apt-get install -y kubelet kubeadm``` par ```sudo apt-get install -y kubelet=VERSION kubeadm=VERSION``` où VERSION est la version de Kubernetes que vous souhaitez installer (par exemple: 1.18.8)
- si vous avez utilisé Multipass pour la création de vos machines virtuelles, vous pouvez obtenir un shell dans une de ces VMs avec la commande ```$ multipass shell NODE_NAME```

```
# Installation de Docker
curl -sSL https://get.docker.com | sh

# Packages nécessaires pour installer Kubernetes via kubeadm
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update && sudo apt-get install -y kubelet kubeadm
```

## Initialisation du cluster

Lancer la commande suivante afin d'initialiser le cluster depuis la première machine (node1):

```
$ sudo kubeadm init
```

Attention: si votre machine n'a pas les ressources CPU nécessaires, il est possible que vous obteniez l'erreur suivante:
```
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR NumCPU]: the number of available CPUs 1 is less than the required 2
```

Dans ce cas vous pouvez relancer la commande d'initialisation avec un paramètre permettant d'ignorer cette erreur (ce n'est cependant pas une solution qui devra être utilisée pour la mise en place d'un cluster de production)

```
$ sudo kubeadm init --ignore-preflight-errors=NumCPU,Mem
```

La mise en place de l'ensemble des composants du master prendra quelques minutes. Vous obtiendrez alors la commande à lancez depuis les autres VMs afin de les ajouter au cluster:

Exemple de commande renvoyée:
```
...
kubeadm join 192.168.33.10:6443 --token i2k2qo.jxqc9nbf2h73ebvj \
    --discovery-token-ca-cert-hash sha256:5bafbd2f2fd318ffeee4f3440c596fecba434432d9d62d43fe308e84899ed07a
```

## Ajout de nodes

Vous devrez ensuite lancer la commande *kubeadm join* obtenue précédemment sur chacune des machines virtuelles afin de les ajouter au cluster (assurez-vous de lancer cette commande en *sudo*):

Exemple de résultat de cette commande lancée depuis le *node2*:
```
$ sudo kubeadm join 192.168.33.10:6443 --token i2k2qo.jxqc9nbf2h73ebvj \
    --discovery-token-ca-cert-hash sha256:5bafbd2f2fd318ffeee4f3440c596fecba434432d9d62d43fe308e84899ed07a
...
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.
...
```

Vous obtiendrez rapidement une confirmation indiquant que la VM fait à présent bien partie du cluster.

Note: si vous avez perdu la commande d'ajout de node, vous pouvez la regénérer avec la commande suivante (à lancer depuis le node master)

```
$ sudo kubeadm token create --print-join-command
```

## Récupération du context

Afin de pouvoir dialoguer avec le cluster via le binaire *kubectl* que vous avez installé sur votre machine locale, il est nécessaire de récupérer le fichier de configuration généré lors de l'installation.

Pour cela, il faut donc récupérer le fichier */etc/kubernetes/admin.conf* présent sur le node master et le copier sur votre machine locale.

Exemple de récupération du fichier de configuration
```
$ scp USER@$IP_DU_NODE_MASTER:/etc/kubernetes/admin.conf do-kube-config
```

Note: si vous avez utilisé *Multipass* pour mettre en place votre cluster, vous pouvez récupérer le fichier de configuration avec la commande suivante (il sera alors sauvegardé dans le fichier *kubeconfig* du répertoire courant):

```
$ multipass exec node1 -- sudo cat /etc/kubernetes/admin.conf > kubeconfig
```

Une fois que le fichier est présent en local, il faut simplement indiquer à *kubectl* ou il se trouve en positionnant la variable d'environnement *KUBECONFIG*:

```
$ export KUBECONFIG=$PWD/kubeconfig
```

Listez alors les nodes du cluster, ils apparaitront avec le status *NotReady*.

```
$ kubectl get nodes
NAME    STATUS     ROLES                  AGE     VERSION
node1   NotReady   control-plane,master   4m45s   v1.20.4
node2   NotReady   <none>                 54s     v1.20.4
node3   NotReady   <none>                 55s     v1.20.4
```

## Installation d'un addons pour le networking entre les Pods

La commande suivante permet d'installer les composants nécessaires pour la communication entre les Pods qui seront déployés sur le cluster:

```
$ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

Vous verrez que plusieurs ressources sont alors créés pour mettre en place cette solution de networking:
```
serviceaccount/weave-net created
clusterrole.rbac.authorization.k8s.io/weave-net created
clusterrolebinding.rbac.authorization.k8s.io/weave-net created
role.rbac.authorization.k8s.io/weave-net created
rolebinding.rbac.authorization.k8s.io/weave-net created
daemonset.apps/weave-net created
```

Note: il y a plusieurs solutions de networking qui peuvent être utilisées, la solution envisagée ici est Weave Net. L'article suivant donne une bonne comparaison des solutions les plus utilisées: [https://objectif-libre.com/fr/blog/2018/07/05/comparatif-solutions-reseaux-kubernetes/](https://objectif-libre.com/fr/blog/2018/07/05/comparatif-solutions-reseaux-kubernetes/), celui-ci effectue un benchmark des différentes solutions [https://itnext.io/benchmark-results-of-kubernetes-network-plugins-cni-over-10gbit-s-network-updated-august-2020-6e1b757b9e49](https://itnext.io/benchmark-results-of-kubernetes-network-plugins-cni-over-10gbit-s-network-updated-august-2020-6e1b757b9e49)

## Vérification de l'état de santé des nodes

Maintenant que la solution de networking a été installée, les nodes devraient être dans l'état *Ready* (il est possible que cela prennent quelques dizaines de secondes)

```
$ kubectl get nodes
NAME    STATUS   ROLES                  AGE     VERSION
node1   Ready    control-plane,master   7m35s   v1.20.4
node2   Ready    <none>                 3m44s   v1.20.4
node3   Ready    <none>                 3m45s   v1.20.4
```

Le cluster est maintenant prêt à être utilisé.

Note: le cluster que vous avez mis en place dans cet exercice contient un node master et 2 nodes worker. Il est également possible avec kubeadm de mettre en place un cluster HA avec plusieurs nodes master en charge de la gestion de l'état du cluster. Kubeadm est très utilisé pour l'installation et la gestion de cluster en environnement de production.
