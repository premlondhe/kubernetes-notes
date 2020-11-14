##### Documentation Link:

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

##### Step 1: Configure Docker
https://kubernetes.io/docs/setup/production-environment/container-runtimes/
```sh
apt-get update
apt-get -y install apt-transport-https ca-certificates curl  gnupg-agent software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

apt-get update && apt-get -y install docker-ce docker-ce-cli
systemctl start docker
```

##### Step 2: Kernel Parameter Configuration
```sh
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

##### Step 3:Configuring Repo and Installation
```sh
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

##### Step 4: Initialize Cluster with kubeadm
This command will provide you two more steps a. Setup config file b. Command to join worker node to the cluster.
```sh
kubeadm init --pod-network-cidr=10.244.0.0/16
```

```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config 
sudo chown $(id -u):$(id -g) $HOME/.kube/config 

kubeadm join 10.240.0.7:6443 --token hwu7xr.gbox0rys7x1pbzag \
     --discovery-token-ca-cert-hash sha256:1c50a509344c41178eac9b5a79849c3141279c3ac0f2842ffc74ffec82847a94
```

##### Step 5: Install Network Addon (flannel)
On master node now when you run "kubectl get nodes" you will see the status "NotReady", hence you have to execute below command on master. Hint: In kubernetes.io, under search type "kube-flannel.yml".

```sh
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
