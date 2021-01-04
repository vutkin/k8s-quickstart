https://kubernetes.io/blog/2019/03/15/kubernetes-setup-using-ansible-and-vagrant/

### Operations
#### Vagrant cli
```
vagrant up
vagrant ssh k8s-master
```

### Dashboard
#### Certs for dashboard

```bash
mkdir certs

openssl req -nodes -newkey rsa:2048 -keyout certs/dashboard.key -out certs/dashboard.csr -subj "/CN=kubernetes-dashboard"
openssl x509 -req -sha256 -days 365 -in certs/dashboard.csr -signkey certs/dashboard.key -out certs/dashboard.crt

kubectl create ns kubernetes-dashboard
kubectl create secret generic kubernetes-dashboard-certs --from-file=certs -n kubernetes-dashboard
```

#### Deploy dashboard
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc7/aio/deploy/recommended.yaml
kubectl -n kubernetes-dashboard patch svc kubernetes-dashboard --type='json' -p '[{"op":"replace","path":"/spec/type","value":"NodePort"}]'
kubectl get svc kubernetes-dashboard -n kubernetes-dashboard -o=jsonpath='{.spec.ports[0].nodePort}{"\n"}'
```

```bash
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
EOF

cat <<EOF | kubectl create -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
EOF

kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

### Deploy nginx
```bash
kubectl create ns test
kubectl create deployment nginx --image=nginx -n test
kubectl create service nodeport nginx --tcp=80:80 -n test
kubectl get svc nginx -n test -o=jsonpath='{.spec.ports[0].nodePort}{"\n"}'
```

### Deploy a hello, world app
```bash
kubectl create ns test
kubectl run web --image=gcr.io/google-samples/hello-app:1.0 --port=8080 -n test
kubectl expose deployment web --target-port=8080 --type=NodePort -n test
kubectl get svc web -n test -o=jsonpath='{.spec.ports[0].nodePort}{"\n"}'
```

### Ingress

```yaml
cat <<EOF | kubectl create -f -
apiVersion: networking.k8s.io/v1beta1 # for versions before 1.14 use extensions/v1beta1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
 rules:
 - host: hello-world.info
   http:
     paths:
     - path: /(.+)
       backend:
         serviceName: web
         servicePort: 8080
EOF
```

### Known issues
#### Flannel minutiae
The VMs in my initial setup had the first NIC connected to the VirtualBox NAT network, and the second NIC connected to a host-only network. This resulted in the pod IPs being inaccessible across nodes — because flannel used the first NIC by default and in my case it was a NAT interface on which the machines could not cross-talk, only reach the web. Thus the pod network did not really work across nodes. An easy way to find out which NIC flannel uses is to look at flannel’s logs:

```yaml
$ kubectl get pod --all-namespaces | grep flannel
kube-system   kube-flannel-ds-j587n          1/1       Running    2          3h
kube-system   kube-flannel-ds-p6sm7          1/1       Running   1          3h
kube-system   kube-flannel-ds-xn27c          1/1       Running   1          3h
$ kubectl logs  -f kube-flannel-ds-j587n -n kube-system
I0622 14:17:37.841808       1 main.go:487] Using interface with name enp0s3 and address 10.0.2.2
```

In the above case you can see that the flannel pod is listening on the NAT interface and this would not work. The way I fixed it was by:
1. Deleting the flannel daemon set:
    ```bash
    kubectl delete ds -l app=flannel -n kube-system
    ```
1. Removing the flannel.1 virtual NIC on each node.
    ```bash
    ip link delete flannel.1
    ```
1. Recreating the flannel daemonset with a tweak in the flannel configmap. Open the `flannel-<...>.yaml` file and append the following tokens to the command directive for thekube-flannel container: `--iface` and `eth1` (or whatever is the name of the NIC connected to the host-only network).
    ```yaml
    args:
      - --ip-masq
      - --kube-subnet-mgr
      - --iface
      - eth1
    ```
1. Finally restarting kubelet and docker on all nodes:
    ```bash
    systemctl stop kubelet && systemctl restart docker && systemctl start kubelet
    ```
[soultion was found in reference article](https://medium.com/@ErrInDam/taming-kubernetes-for-fun-and-profit-60a1d7b353de)

### Ingress controllers
```bash
kubectl apply -f https://raw.githubusercontent.com/haproxytech/kubernetes-ingress/master/deploy/haproxy-ingress.yaml
```

### Assign to master node
```yaml
nodeSelector:
  node-role.kubernetes.io/master: ""
```
### Controlling your cluster from machines other than the control-plane node
```bash
scp root@<master ip>:/etc/kubernetes/admin.conf .
kubectl --kubeconfig ./admin.conf get nodes
```
#### Proxying API Server to localhost
```bash
kubectl --kubeconfig ./admin.conf proxy
```
