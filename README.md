### Prepare an OCM cluster

### Install OCM Cluster-Proxy Addon

1. Install helm chart in the hub cluster:

```shell
$ helm repo add cluster-proxy https://github.com/yue9944882/cluster-proxy/releases/download/v0.0.10/
$ helm repo update
$ helm install \
    -n open-cluster-management-cluster-proxy \
    --create-namespace  \
    cluster-proxy cluster-proxy/cluster-proxy 
```

2. Install addon for the managed cluster (replace your cluster name in <...>):

```
$ kubectl create -f <<<EOF
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: cluster-proxy
  namespace: <your cluster name>
spec:
  installNamespace: "open-cluster-management-cluster-proxy"
EOF
```

3. Check if the addon is properly installed:

```shell
$ kubectl -n <your cluster name> get managedclusteraddon
NAME                     AVAILABLE   DEGRADED   PROGRESSING
cluster-proxy            True                   
```

### Install OCM ManagedServiceAccount Addon

1. Install helm chart in the hub cluster:

```shell
$ helm repo add managed-serviceaccount https://github.com/yue9944882/managed-serviceaccount/releases/download/v0.0.24/
$ helm repo update
$ helm install \
    -n open-cluster-management-managed-serviceaccount \
    --create-namespace  \
    managed-serviceaccount managed-serviceaccount/managed-serviceaccount
```


2. Install addon for the managed cluster (replace your cluster name in <...>):

```shell
$ kubectl create -f <<<EOF
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: managed-serviceaccount
  namespace: <your cluster name>
spec:
  installNamespace: "open-cluster-management-managed-serviceaccount"
EOF
```

3. Check if the addon is properly installed:

```shell
$ kubectl -n <your cluster name> get managedclusteraddon
NAME                     AVAILABLE   DEGRADED   PROGRESSING
managed-serviceaccount   True                   
```

4. Create a `ManagedServiceAccount` into the hub cluster to project the token:

```shell
$ kubectl create -f <<<EOF
apiVersion: authentication.open-cluster-management.io/v1alpha1
kind: ManagedServiceAccount
metadata:
  name: cluster-gateway
  namespace: <your cluster name>
spec:
  projected:
    secret:
      labels:
        cluster.core.oam.dev/cluster-credential-type: ServiceAccountToken
        cluster.core.oam.dev/cluster-endpoint-type: ClusterProxy
      name: <your cluster name>
      namespace: open-cluster-management-credentials
    type: Secret
  rotation:
    enabled: true
    validity: 8640h0m0s
EOF
```

5. Check if the addon is properly installed by reading the status:

```shell
$ kubecl -n <your cluster name> get managedserviceaccount cluster-gateway -o yaml
```

### Install OCM ClusterGateway Addon

2. Copying cluster-proxy client secrets to the cluster-gateway's namespace

```shell
$ kubectl create ns  open-cluster-management-cluster-gateway
$ kubectl get secret proxy-client -n open-cluster-management-cluster-proxy  -o yaml | sed 's/namespace: .*/namespace: open-cluster-management-cluster-gateway/' | kubectl apply -f -
$ kubectl get secret proxy-server-ca -n open-cluster-management-cluster-proxy  -o yaml | sed 's/namespace: .*/namespace: open-cluster-management-cluster-gateway/' | kubectl apply -f -
```
   
2. Install helm chart in the hub cluster:

```shell
$ helm repo add cluster-gateway https://github.com/oam-dev/cluster-gateway/releases/download/v1.1.6/
$ helm repo update
$ helm install \
    -n open-cluster-management-cluster-gateway \
    --create-namespace  \
    cluster-gateway cluster-gateway/cluster-gateway 
```

3. Try probing the cluster via gateway:

```shell
$ kubectl get --raw="/apis/cluster.core.oam.dev/v1alpha1/clustergateways/<your cluster name>/proxy/healthz"
```

