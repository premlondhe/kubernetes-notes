#### Pre-Requisites:
```sh
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
yum -y install docker && systemctl start docker && systemctl enable docker
yum -y install socat conntrack ipset
sysctl -w net.ipv4.conf.all.forwarding=1
cd  /root/binaries/kubernetes/node/bin/
cp kube-proxy kubectl kubelet /usr/local/bin
```

#### Step 1: Generate Kubelet Certificate for Worker Node.

Note:
   1. Replace the IP Address and Hostname field in the below configurations according to your enviornement.
   2. Run this in the Kubernetes Master Node
   
**NOTE:** Run ```hostname``` command on worker node to get **WORKER-NODE-HOSTNAME**. **WORKER-IP1** & **WORKER-IP2** are the Public/Private ips of Azure/AWS VM    

**ON Master Node**
```sh
WORKER_HOSTNAME="WORKER-NODE-HOSTNAME"
```

**ON Worker Node**
```sh
WORKER_HOSTNAME="WORKER-NODE-HOSTNAME"
```

```sh
cd /root/certificates
```
```sh
cat > openssl-${WORKER_HOSTNAME}.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = ${WORKER_HOSTNAME}
IP.1 = WORKER-IP1
IP.2 = WORKER-IP2
EOF
```
```sh
openssl genrsa -out ${WORKER_HOSTNAME}.key 2048
```
```sh
openssl req -new -key ${WORKER_HOSTNAME}.key -subj "/CN=system:node:${WORKER_HOSTNAME}/O=system:nodes" -out ${WORKER_HOSTNAME}.csr -config openssl-${WORKER_HOSTNAME}.cnf
openssl x509 -req -in ${WORKER_HOSTNAME}.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out ${WORKER_HOSTNAME}.crt -extensions v3_req -extfile openssl-${WORKER_HOSTNAME}.cnf -days 1000
```

#### Step 2: Generate kube-proxy certificate:
```sh
openssl genrsa -out kube-proxy.key 2048
openssl req -new -key kube-proxy.key -subj "/CN=system:kube-proxy" -out kube-proxy.csr
openssl x509 -req -in kube-proxy.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-proxy.crt -days 1000
```
#### Step 3: Copy Certificates to Worker Node:

This can either be manual approach or via SCP.
Certificates: kubelet, kube-proxy and CA certificate.

In-case you want to automate it, then following configuration can be used.
In the demo, we had made used of manual way.

In-case, you want to transfer file from master to worker node, then you can make use of the following approach:

- Worker Node:
```sh
nano /etc/ssh/sshd_config
PasswordAuthentication yes
systemctl restart sshd
useradd zeal
passwd zeal5872#
```
- Master Node:
```sh
scp kube-proxy.crt kube-proxy.key ${WORKER_HOSTNAME}.crt ${WORKER_HOSTNAME}.key ca.crt zeal@161.35.205.5:/tmp

```
- Worker Node:
```sh
cd /tmp
mv kube-proxy.crt kube-proxy.key ${WORKER_HOSTNAME}.crt ${WORKER_HOSTNAME}.key ca.crt /root/certificates

```
#### Step 4: Move Certificates to Specific Location.
```sh
cd /root/certificates
mkdir /var/lib/kubernetes
cp ca.crt /var/lib/kubernetes
mkdir /var/lib/kubelet
mv ${WORKER_HOSTNAME}.crt  ${WORKER_HOSTNAME}.key  kube-proxy.crt  kube-proxy.key ca.crt /var/lib/kubelet/
```
#### Step 5: Generate Kubelet Configuration YAML File:
```sh
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.crt"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
runtimeRequestTimeout: "15m"
EOF
```
#### Step 6: Generate Systemd service file for kubelet:

Create Systemd file for Kubelet:
```sh
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --tls-cert-file=/var/lib/kubelet/${WORKER_HOSTNAME}.crt \\
  --tls-private-key-file=/var/lib/kubelet/${WORKER_HOSTNAME}.key \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2 \\
  --cgroup-driver=systemd \\
  --runtime-cgroups=/systemd/system.slice \\
  --kubelet-cgroups=/systemd/system.slice
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
#### Step 7: Generate the Kubeconfig file for Kubelet

```sh
cd /var/lib/kubelet
SERVER_IP=IP-OF-API-SERVER
```
```sh
{
  kubectl config set-cluster kubernetes-from-scratch \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://${SERVER_IP}:6443 \
    --kubeconfig=${WORKER_HOSTNAME}.kubeconfig

  kubectl config set-credentials system:node:${WORKER_HOSTNAME} \
    --client-certificate=${WORKER_HOSTNAME}.crt \
    --client-key=${WORKER_HOSTNAME}.key \
    --embed-certs=true \
    --kubeconfig=${WORKER_HOSTNAME}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-from-scratch \
    --user=system:node:${WORKER_HOSTNAME} \
    --kubeconfig=${WORKER_HOSTNAME}.kubeconfig

  kubectl config use-context default --kubeconfig=${WORKER_HOSTNAME}.kubeconfig
}
```
```sh
mv ${WORKER_HOSTNAME}.kubeconfig kubeconfig
```
### Part 2 - Kube-Proxy

#### Step 1: Copy Kube Proxy Certificate to Directory:
```sh
mkdir /var/lib/kube-proxy

```
#### Step 2: Generate KubeConfig file:
```sh
{
  kubectl config set-cluster kubernetes-from-scratch \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://${SERVER_IP}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.crt \
    --client-key=kube-proxy.key \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-from-scratch \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
}
```
```sh
mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```
#### Step 3: Generate kube-proxy configuration file:
```sh
cd /var/lib/kube-proxy
```
```sh
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```
#### Step 4: Create kube-proxy service file:
```sh
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

#### Step 4.1: Disable swap if its on
```sh 
swapon -s
swapoff /partition/name
```


#### Step 5:
```sh
systemctl start kubelet
systemctl start kube-proxy
systemctl enable kubelet
systemctl enable kube-proxy
```

#### Step 6: On Master node check node is visible.

```sh 
kubectl get nodes
```
```sh
root@kube-prem-master1:~/certificates# kubectl get nodes
NAME                STATUS     ROLES    AGE    VERSION
kube-prem-worker1   NotReady   <none>   119s   v1.19.0
root@kube-prem-master1:~/certificates# kubectl get nodes
NAME                STATUS     ROLES    AGE     VERSION
kube-prem-worker1   NotReady   <none>   4m48s   v1.19.0
```

