# Kubernetes Security Hands-On

## Minikube

Start Minikube

```bash
minikube delete

# for hyperkit (or other vm drivers)
minikube config set vm-driver hyperkit

minikube start --network-plugin=cni --memory=4096

# Install Cilium (for Network Policies later)
kubectl create -f https://raw.githubusercontent.com/cilium/cilium/v1.5/examples/kubernetes/1.14/cilium-minikube.yaml
```

## RBAC

### User RBAC

```bash
kubectl create namespace rbac-example
kubectl create serviceaccount -n rbac-example myuser
kubectl create rolebinding -n rbac-example myuser-view --clusterrole=view --serviceaccount=rbac-example:myuser

alias kubectl-user='kubectl --as=system:serviceaccount:rbac-example:myuser'

kubectl-user get pod -n rbac-example
kubectl-user get pod
kubectl get pod
kubectl-user auth can-i get pods -n default
kubectl create rolebinding -n default myuser-default-view --clusterrole=view --serviceaccount=rbac-example:myuser
kubectl-user auth can-i get pods -n default
kubectl-user get pod
kubectl-user auth can-i get pods --all-namespaces
```

#### Some examples

Admin access to a specific namespace:

```YAML
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: development-admin
  namespace: development
subjects:
  - kind: User
    name: jane
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: dev-admin
    apiGroup: rbac.authorization.k8s.io  
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
```

Read access to the whole cluster:

```YAML
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cluster-viewer
subjects:
  - kind: User
    name: jane
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: cluster-view
    apiGroup: rbac.authorization.k8s.io  
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

### Workload RBAC

```bash
kubectl create -f prometheus.yaml
kubectl -n kube-system get pods
kubectl -n kube-system logs prometheus-0
vi prometheus.yaml
kubectl apply -f prometheus.yaml
kubectl create -f prometheus-rbac.yaml
kubectl -n kube-system delete pod prometheus-0
kubectl -n kube-system get pods
kubectl -n kube-system logs prometheus-0
```

## Network Policies

### Ingress Policies

```bash
kubectl create ns restricted
kubectl run -n restricted --image=nginx nginx-app --port=80
kubectl -n restricted get pod -o wide
kubectl run utils \
  --restart Never \
  --image webwurst/curl-utils \
  --command sleep 3000
kubectl exec utils curl IPOFNGINX:80

# Deny all (ingress) traffic to pods in that namespace
kubectl create -n restricted -f default-deny.yaml

kubectl exec utils curl IPOFNGINX:80

# Allow traffic from busybox to nginx
kubectl label ns default name=default
kubectl create -n restricted -f allow-nginx.yaml

kubectl exec utils curl IPOFNGINX:80

kubectl -n restricted run bla \
  --restart Never \
  --image webwurst/curl-utils \
  --command sleep 3000
kubectl -n restricted exec bla curl IPOFNGINX:80

# Allow all traffic within namespace
label ns restricted name=restricted
kubectl create -f allow-within-ns.yaml

kubectl -n restricted exec bla curl IPOFNGINX:80
```

### Egress Policies

Egress to pods within a cluster

```bash
kubectl -n restricted exec bla nslookup google.de

# Deny all egress in namespace
kubectl -n restricted create default-deny-egress.yaml

kubectl -n restricted exec bla nslookup google.de

# Allow DNS lookups
kubectl label ns kube-system name=kube-system
kubectl -n restricted create allow-dns.yaml

kubectl -n restricted exec bla nslookup google.de
```

Egress to IPs outside the cluster

```bash
kubectl -n restricted exec bla ping 8.8.8.8

# Allow
kubectl -n restricted create allow-external.yaml

kubectl -n restricted exec bla ping 8.8.8.8
```

## PSP

Running minikube with PSP is not trivial, you can start it by running

```bash
minikube start \
  --extra-config=apiserver.enable-admission-plugins="NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,PodSecurityPolicy,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota"
```

This is will take a lot of time as minikube wants to verify it is working. 
It will finally result in a failed start, but minikube should actually be running. 
Just the Kubernetes components (besides API server) won't be up. 
To get them running you can apply following manigest that contain default PSPs and 
bindings for the main components.

```bash
kubectl create minikube-psp.yaml
```

Once you have a working minikube with PSP enabled you can check out https://kubernetes.io/docs/concepts/policy/pod-security-policy/#example and https://docs.giantswarm.io/guides/securing-with-rbac-and-psp/#running-applications-that-need-privileged-access.

## Sources, Further Reading and Resources

- https://docs.giantswarm.io/guides/securing-with-rbac-and-psp/
- https://github.com/appscodelabs/tasty-kube/tree/master/minikube/1.10/psp
- https://kubernetes.io/docs/concepts/policy/pod-security-policy/#example
- https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/prometheus
- https://docs.cilium.io/en/v1.5/gettingstarted/minikube/
- https://kubernetes.io/docs/concepts/services-networking/network-policies/