
#### Step 1: Verify the status of DNS
```sh
kubectl cluster-info
```
#### Step 2: Create a BusyBox POD to test DNS
```sh
kubectl run busybox --image=busybox:1.28 --command -- sleep 3600
```
```sh
kubectl exec -it busybox -- sh
nslookup google.com
ping 8.8.8.8
```
#### Step 3: Install CoreDNS: Open below URL in browser and save on your master node with name ```coredns.yaml```.
```sh
browser:> https://github.com/premlondhe/kubernetes-notes/blob/master/Domain%206%20-%20Cluster%20Architecture%2C%20Installation%20%26%20Configuration/coredns.yaml
```
```sh
kubectl apply -f coredns.yaml
```

#### Step 4: Verify if DNS Resolution Works:

```sh
kubectl get pods -n kube-system
kubectl exec -it busybox -- sh
nslookup google.com
nslookup kubernetes
```
