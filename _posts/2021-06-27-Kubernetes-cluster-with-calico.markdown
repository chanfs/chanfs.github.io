---
layout: post
title: Kubernetes cluster with Calico
date: 2021-06-27 01:47:00 +0800
categories: Container Kubernetes Calico
---

## How to setup a 3 node Kubernetes cluster with Calico using kubeadm for production environment

The last time I run Kubernetes is in 2018. It was for a project that require NFV following Intel release of the Multus CNI Plugin that allows K8s pods to be multi-homed. Please refer to 
[https://builders.intel.com/docs/networkbuilders/enabling_new_features_in_kubernetes_for_NFV.pdf](https://builders.intel.com/docs/networkbuilders/enabling_new_features_in_kubernetes_for_NFV.pdf)

This time I would like to try the Calico pod network. I hope explore the K8s network policy in the future and Calico supports it.

# The setup
k1 - Control plane node

kn1, kn2 - worker nodes

All 3 nodes are running Ubuntu 18 on Virtualbox and they are all in the same subnet (192.168.1.0/24).

Pod network: 192.168.2.0/24

Container runtime: Docker

# Host file or DNS
As I plan to use ```--control-plane-endpoint``` during kubeadm init so that I can load balance the control plane in the future, it is important to setup the host file or DNS so that all nodes can resolve to the control plane node. 

For my case i add the hostname *cluster-endpoint* to the host files for all nodes.


# The NIC issue
Usually I prefer to use 2 NICs for my VMs on Virtualbox, one NAT and one Host-Only, but I soon run into problems during ```kubeadm init``` as it uses the NAT NIC since there is a default route for it. So I change to use only one NIC on the bridge interface. But I believe the issue can be avoided by using ``` --apiserver-advertise-address ``` during ```kubeadm init```. 

# Docker
I installed Docker during Ubuntu installation but I encountered issue during the Cgroup drivers configuration when i run :
```
sudo systemctl enable docker

sudo systemctl daemon-reload

sudo systemctl restart docker
```
So I chose to install Docker manually following [https://docs.docker.com/engine/install/ubuntu/](https://docs.docker.com/engine/install/ubuntu/) using the repository method.

# Installing kubeadm
It is quite straightforward, although you really need to slow down and read everything, especially if you have installed Kubernetes cluster using kubeadm before, as you may think you know and overlook some steps, which actually happened to me.

Follow the instructions in [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

# Creating a cluster with kubeadm


```fs@k1:~$ sudo kubeadm init --control-plane-endpoint=cluster-endpoint --pod-network-cidr=192.168.2.0/24```

```console
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join cluster-endpoint:6443 --token req5hj.jau914fncccdb5q5 \
        --discovery-token-ca-cert-hash sha256:aaaa5d2e7aeffa24650dc5c229f396aa4354578bb9cba2b908c1e52953435a97 \
        --control-plane

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join cluster-endpoint:6443 --token req5hj.jau914fncccdb5q5 \
        --discovery-token-ca-cert-hash sha256:aaaa5d2e7aeffa24650dc5c229f396aa4354578bb9cba2b908c1e52953435a97
```
On the control plane node, i run this
```shell
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

# Install Calico pod network
On the control plane node
```shell
fs@k1:~$ curl https://docs.projectcalico.org/manifests/calico.yaml -O
```
As i want to use 192.168.2.0/24 as my pod network, i edited calico.yaml according to the docs at https://docs.projectcalico.org/getting-started/kubernetes/self-managed-onprem/onpremises
which mentioned:"If you are using pod CIDR 192.168.0.0/16, skip to the next step. If you are using a different pod CIDR with kubeadm, no changes are required - Calico will automatically detect the CIDR based on the running configuration. **For other platforms, make sure you uncomment the CALICO_IPV4POOL_CIDR variable in the manifest and set it to the same value as your chosen pod CIDR."**

So I edit the file to include:

 - name: CALICO_IPV4POOL_CIDR
    value: "192.168.2.0/24"

but it is not working

~~~console
fs@k1:~$ sudo kubectl apply -f calico.yaml
configmap/calico-config created
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
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
error: error parsing calico.yaml: error converting YAML to JSON: yaml: line 182: did not find expected '-' indicator

fs@k1:~$ kubectl get pods --all-namespaces
error: error loading config file "/home/fs/.kube/config": open /home/fs/.kube/config: permission denied
fs@k1:~$ sudo kubectl get pods --all-namespaces
NAMESPACE     NAME                         READY   STATUS    RESTARTS   AGE
kube-system   coredns-558bd4d5db-7dkks     0/1     Pending   0          23h
kube-system   coredns-558bd4d5db-8tcng     0/1     Pending   0          23h
kube-system   etcd-k1                      1/1     Running   1          23h
kube-system   kube-apiserver-k1            1/1     Running   1          3d12h
kube-system   kube-controller-manager-k1   1/1     Running   1          23h
kube-system   kube-proxy-2wkv4             1/1     Running   1          23h
kube-system   kube-scheduler-k1            1/1     Running   1          3d12h
~~~

So it does not need modification

~~~console
fs@k1:~$ curl https://docs.projectcalico.org/manifests/calico.yaml -O

fs@k1:~$ sudo kubectl apply -f calico.yaml

configmap/calico-config unchanged
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org configured
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers unchanged
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers unchanged
clusterrole.rbac.authorization.k8s.io/calico-node unchanged
clusterrolebinding.rbac.authorization.k8s.io/calico-node unchanged
daemonset.apps/calico-node created
serviceaccount/calico-node created
deployment.apps/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
Warning: policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
poddisruptionbudget.policy/calico-kube-controllers created

fs@k1:~$ sudo kubectl get pods --all-namespaces
NAMESPACE     NAME                                       READY   STATUS     RESTARTS   AGE
kube-system   calico-kube-controllers-78d6f96c7b-dbmsc   0/1     Pending    0          8s
kube-system   calico-node-rdqkm                          0/1     Init:0/3   0          8s
kube-system   coredns-558bd4d5db-7dkks                   0/1     Pending    0          23h
kube-system   coredns-558bd4d5db-8tcng                   0/1     Pending    0          23h
kube-system   etcd-k1                                    1/1     Running    1          23h
kube-system   kube-apiserver-k1                          1/1     Running    1          3d12h
kube-system   kube-controller-manager-k1                 1/1     Running    1          23h
kube-system   kube-proxy-2wkv4                           1/1     Running    1          23h
kube-system   kube-scheduler-k1                          1/1     Running    1          3d12h

fs@k1:~$ sudo kubectl get pods --all-namespaces
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-78d6f96c7b-dbmsc   1/1     Running   0          77s
kube-system   calico-node-rdqkm                          1/1     Running   0          77s
kube-system   coredns-558bd4d5db-7dkks                   1/1     Running   0          23h
kube-system   coredns-558bd4d5db-8tcng                   1/1     Running   0          23h
kube-system   etcd-k1                                    1/1     Running   1          23h
kube-system   kube-apiserver-k1                          1/1     Running   1          3d12h
kube-system   kube-controller-manager-k1                 1/1     Running   1          23h
kube-system   kube-proxy-2wkv4                           1/1     Running   1          23h
kube-system   kube-scheduler-k1                          1/1     Running   1          3d12h
~~~

# Join worker node *kn2* to the cluster
~~~console
fs@k1:~$ sudo kubeadm token create
vilx4e.9jl70x8ixvcqantz

fs@k1:~$ sudo kubeadm token list
TOKEN                     TTL         EXPIRES                USAGES                   DESCRIPTION                                                EXTRA GROUPS
vilx4e.9jl70x8ixvcqantz   23h         2021-06-17T16:51:22Z   authentication,signing   <none>                                                     system:bootstrappers:kubeadm:default-node-token
fs@k1:~$


fs@kn2:~$ sudo kubeadm join cluster-endpoint:6443 --token vilx4e.9jl70x8ixvcqantz         --discovery-token-ca-cert-hash sha256:a0d65d2e7aeffa24650dc5c229f396aa4354578bb9cba2b908c1e52953435a97
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

fs@kn2:~$


fs@k1:~$ kubectl get nodes
error: error loading config file "/home/fs/.kube/config": open /home/fs/.kube/config: permission denied
fs@k1:~$ sudo kubectl get nodes
NAME   STATUS     ROLES                  AGE     VERSION
k1     Ready      control-plane,master   3d13h   v1.21.1
kn2    NotReady   <none>                 17s     v1.21.1

fs@k1:~$ sudo kubectl get nodes
NAME   STATUS   ROLES                  AGE     VERSION
k1     Ready    control-plane,master   3d13h   v1.21.1
kn2    Ready    <none>                 4m41s   v1.21.1
~~~
Repeat the same for worker node kn1. There is no need to generate the token as it lives on for 24 hours. Reuse the same token.

# Run a Ngnix deployment on the cluster
On the control plane node
~~~console
wget https://k8s.io/examples/controllers/nginx-deployment.yaml
vi nginx-deployment.yaml
~~~

Change replica to 2 since we only have two worker nodes
~~~vi
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
~~~
Deploy it
~~~console
fs@k1:~$ sudo kubectl apply -f nginx-deployment.yaml
deployment.apps/nginx-deployment created
fs@k1:~$
~~~

Get info on the running pods

~~~console
fs@k1:~$ sudo kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-66b6c48dd5-gxbks   1/1     Running   0          45s
nginx-deployment-66b6c48dd5-wv7wd   1/1     Running   0          45s
fs@k1:~$
~~~

That's it. We have got a production grade Kubernetes cluster running!




