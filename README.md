https://kubernetes.io/blog/2019/03/15/kubernetes-setup-using-ansible-and-vagrant/

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
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta4/aio/deploy/recommended.yaml
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

### Issues
#### flannel network
https://github.com/coreos/flannel/issues/871

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