#### Documentation Referred:

https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/

#### Step 1: Try connecting to Busybox pod:
**Note:** You can comment out entries in ```/etc/hosts``` first, than run below command -
```sh
kubectl exec -it busybox -- sh
```

You will see below error on executing above command -
```sh 
root@kube-prem-master1:~/svc# kubectl exec -it busybox -- sh
Error from server: error dialing backend: dial tcp: lookup kube-prem-worker1 on 127.0.0.53:53: server misbehaving
```

#### Step 2: Installing nslookup utility
```sh
yum -y install bind-utils
nslookup WORKER-NODE-X
```
### Step 2.1: 
In the output, You will see the **Hostname** and **InternalIP** under **Addresses** section, so we will prefer connecting internal IP instead of hostname. 

```sh 
kubectl describe node WORKER-NODE-X
```

#### Step 3: Modify the Configuration file for API Service
```sh
nano /etc/systemd/system/kube-apiserver.service
```

##### Add following flag:

```sh
--kubelet-preferred-address-types InternalIP
```
```sh
systemctl daemon-reload
systemctl restart kube-apiserver
```

#### Step 4: Verifify Connectivity
```sh
kubectl exec -it  busybox -- sh
```
