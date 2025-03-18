# Issue description

gloo-fed stopped watching its resources and reconciles the `GlooInstance` check section only on start.
If you'll run this manual of 1.17.5 it will work just fine.

# How to reproduce

1. Create a target clusters with gloo edge enterprise:

```shell
helm repo add glooe https://storage.googleapis.com/gloo-ee-helm
helm repo update

KUBECONFIG=target-kubeconfig.yaml kind create cluster --name target --config=- << EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 31500
    hostPort: 31500
    protocol: TCP
EOF

helm upgrade --kubeconfig target-kubeconfig.yaml --install gloo glooe/gloo-ee --version 1.18.7 --namespace gloo-system \
  --create-namespace \
  --set-string license_key=${GLOO_LICENSE_KEY} \
  --wait \
  --wait-for-jobs \
  --values - << EOF
gloo:
  gatewayProxies:
    gatewayProxy:
      service:
        type: NodePort
        httpPort: 31500
        httpNodePort: 31500
      podTemplate:
        probes: true
  discovery:
    enabled: false
  gateway:
    validation:
      alwaysAcceptResources: false
global:
  extensions:
    extAuth:
      enabled: false
    rateLimit:
      enabled: false
grafana:
  defaultInstallationEnabled: false
observability:
  enabled: false
gloo-fed:
  enabled: false
  glooFedApiserver:
    enable: false
prometheus:
  enabled: false
EOF

2. Create source cluster with gloo federation:

```shell
export KUBECONFIG=source-kubeconfig.yaml
kind create cluster --name source
helm repo add gloo-fed https://storage.googleapis.com/gloo-fed-helm
helm repo update
helm upgrade --install gloo-fed gloo-fed/gloo-fed \
  --version 1.18.7 \
  --namespace gloo-system \
  --create-namespace \
  --set-string license_key=${GLOO_LICENSE_KEY} \
  --wait \
  --values - << EOF
glooFedApiserver:
  enable: false
glooFed:
  retries:
    clusterWatcherRemote:
      type: fixed
      delay: 5s
      maxJitter: 100ms
      attempts: 0
    clusterWatcherLocal:
      type: fixed
      delay: 1s
      maxJitter: 100ms
      attempts: 0
EOF
```

3. Create kubeconfig secrets in source cluster for the target cluster:

```shell
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
type: solo.io/kubeconfig
metadata:
  name: target
  namespace: gloo-system
data:
  kubeconfig: $(kind get kubeconfig --name target --internal | base64 | tr -d '[:space:]')
EOF
```

4. Create `KubernetesCluster` resource for the target cluster:

```shell
cat <<EOF | kubectl apply -f -
apiVersion: multicluster.solo.io/v1alpha1
kind: KubernetesCluster
metadata:
  name: target
  namespace: gloo-system
spec:
  secretName: target
EOF
```

5. Observe the `GlooInstance` resource which should reflect the currect status in the target cluster:
```shell
kubectl get glooinstances -n gloo-system target-gloo-system
```

8. Create `FederatedGateway` resource, delete a `Gateway` from the target cluster, scale dowm `Gloo` / `GatewayProxy` deployments and notice the `GlooInstance` isn't updated

9. Restart the gloo-fed pod and see that it updates the `check` section in the `GlooInstance`
