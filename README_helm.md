# Solo.io Istio Ambient Multi-cluster Workshop

In this workshop, you will set up Istio Ambient in a multi-cluster environment, deploy a sample Bookinfo app, and explore how Solo.io Ambient enables secure service-to-service communication in all directions.

![overview](overview.png)

|[![Gloo Mesh Ambient Multi-Cluster Video](yt-video.png)](https://youtu.be/18dpBukOYSk) |
|:-----------------------------:|
### Env

1. Create two clusters and set the env vars below to their context

```bash
export CLUSTER1=gke_ambient_one # UPDATE THIS
export CLUSTER2=gke_ambient_two # UPDATE THIS
export GLOO_MESH_LICENSE_KEY=<update>  # UPDATE THIS

export ISTIO_VERSION=1.25.2
export REPO_KEY=e038d180f90a
export ISTIO_IMAGE=${ISTIO_VERSION}-solo
export REPO=us-docker.pkg.dev/gloo-mesh/istio-${REPO_KEY}
export HELM_REPO=us-docker.pkg.dev/gloo-mesh/istio-helm-${REPO_KEY}
```
2. Download Solo's 1.25.1 `istioctl` Binary:
```bash
OS=$(uname | tr '[:upper:]' '[:lower:]' | sed -E 's/darwin/osx/')
ARCH=$(uname -m | sed -E 's/aarch/arm/; s/x86_64/amd64/; s/armv7l/armv7/')

mkdir -p ~/.istioctl/bin
curl -sSL https://storage.googleapis.com/istio-binaries-e038d180f90a/${ISTIO_VERSION}-solo/istioctl-${ISTIO_VERSION}-solo-${OS}-${ARCH}.tar.gz | tar xzf - -C ~/.istioctl/bin
chmod +x ~/.istioctl/bin/istioctl

export PATH=${HOME}/.istioctl/bin:${PATH}
```

3. Verify using `istioctl version`

### Clean up previous Istio installations

```
```bash
kubectl --context=$CLUSTER1 delete smc --all
kubectl --context=$CLUSTER2 delete smc --all
istioctl uninstall --purge -y --context $CLUSTER1
istioctl uninstall --purge -y --context $CLUSTER2
```

### Deploy Bookinfo sample to both clusters
```bash
for context in ${CLUSTER1} ${CLUSTER2}; do
  kubectl --context ${context} create ns bookinfo 
  kubectl --context ${context} apply -n bookinfo -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/bookinfo/platform/kube/bookinfo.yaml
  kubectl --context ${context} apply -n bookinfo -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/bookinfo/platform/kube/bookinfo-versions.yaml
done
```

### Configure Trust - Issue Intermediate Certs

```bash
for context in ${CLUSTER1} ${CLUSTER2}; do
  kubectl --context=${context} create ns istio-system || true
  kubectl --context=${context} create ns istio-gateways || true
done
kubectl --context=${CLUSTER1} create secret generic cacerts -n istio-system \
--from-file=./certs/cluster1/ca-cert.pem \
--from-file=./certs/cluster1/ca-key.pem \
--from-file=./certs/cluster1/root-cert.pem \
--from-file=./certs/cluster1/cert-chain.pem
kubectl --context=${CLUSTER2} create secret generic cacerts -n istio-system \
--from-file=./certs/cluster2/ca-cert.pem \
--from-file=./certs/cluster2/ca-key.pem \
--from-file=./certs/cluster2/root-cert.pem \
--from-file=./certs/cluster2/cert-chain.pem
```

GKE Only:
```bash
for context in ${CLUSTER1} ${CLUSTER2}; do
kubectl --context  ${context} -n istio-system apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: gcp-critical-pods
  namespace: istio-system
spec:
  hard:
    pods: 1000
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values:
      - system-node-critical
EOF
done
```

Install Istio CRD's on cluster1 and cluster2:

```bash
for context in ${CLUSTER1} ${CLUSTER2}; do
    helm upgrade --install istio-base oci://${HELM_REPO}/base \
    --namespace istio-system \
    --create-namespace \
    --version ${ISTIO_IMAGE} \
    -f - <<EOF
defaultRevision: ""
profile: ambient
EOF
done
```

Apply the CRDs for the Kubernetes Gateway API to your cluster, which are required to create components such as waypoint proxies for L7 traffic policies, gateways with the Gateway resource, and more.

```bash
kubectl --context $CLUSTER1 apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
kubectl --context $CLUSTER2 apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
```

Install istiod on cluster1 and cluster2
```bash
helm upgrade --install istiod oci://${HELM_REPO}/istiod \
--namespace istio-system \
--kube-context ${CLUSTER1} \
--version ${ISTIO_IMAGE} \
-f - <<EOF
env:
  # Assigns IP addresses to multicluster services
  PILOT_ENABLE_IP_AUTOALLOCATE: "true"
  # Disable selecting workload entries for local service routing.
  # Required for Gloo VirtualDestinaton functionality.
  PILOT_ENABLE_K8S_SELECT_WORKLOAD_ENTRIES: "false"
  # Required when meshConfig.trustDomain is set
  PILOT_SKIP_VALIDATE_TRUST_DOMAIN: "true"
global:
  hub: ${REPO}
  multiCluster:
    clusterName: cluster1
  network: cluster1
  proxy:
    clusterDomain: cluster.local
  tag: ${ISTIO_IMAGE}
istio_cni:
  namespace: istio-system
  enabled: true
meshConfig:
  accessLogFile: /dev/stdout
  defaultConfig:
    proxyMetadata:
      ISTIO_META_DNS_AUTO_ALLOCATE: "true"
      ISTIO_META_DNS_CAPTURE: "true"
# Required to enable multicluster support
platforms:
  peering:
    enabled: true
profile: ambient
license:
    value: ${GLOO_MESH_LICENSE_KEY}
EOF
```
```bash
helm upgrade --install istiod oci://${HELM_REPO}/istiod \
--namespace istio-system \
--kube-context ${CLUSTER2} \
--version ${ISTIO_IMAGE} \
-f - <<EOF
env:
  # Assigns IP addresses to multicluster services
  PILOT_ENABLE_IP_AUTOALLOCATE: "true"
  # Disable selecting workload entries for local service routing.
  # Required for Gloo VirtualDestinaton functionality.
  PILOT_ENABLE_K8S_SELECT_WORKLOAD_ENTRIES: "false"
  # Required when meshConfig.trustDomain is set
  PILOT_SKIP_VALIDATE_TRUST_DOMAIN: "true"
global:
  hub: ${REPO}
  multiCluster:
    clusterName: cluster2
  network: cluster2
  proxy:
    clusterDomain: cluster.local
  tag: ${ISTIO_IMAGE}
istio_cni:
  namespace: istio-system
  enabled: true
meshConfig:
  accessLogFile: /dev/stdout
  defaultConfig:
    proxyMetadata:
      ISTIO_META_DNS_AUTO_ALLOCATE: "true"
      ISTIO_META_DNS_CAPTURE: "true"
# Required to enable multicluster support
platforms:
  peering:
    enabled: true
profile: ambient
license:
    value: ${GLOO_MESH_LICENSE_KEY}
EOF
```

Install istio-cni on cluster1 and cluster2. **FOR GKE**, uncomment the `platform: gke` line
```bash
helm upgrade --install istio-cni oci://${HELM_REPO}/cni \
--namespace istio-system \
--kube-context ${CLUSTER1} \
--version ${ISTIO_IMAGE} \
-f - <<EOF
# Assigns IP addresses to multicluster services
ambient:
  dnsCapture: true
excludeNamespaces:
  - istio-system
  - kube-system
global:
  hub: ${REPO}
  tag: ${ISTIO_IMAGE}
  variant: distroless
  # platform: gke # UNCOMMENT FOR GKE
profile: ambient
EOF
```

```bash
helm upgrade --install istio-cni oci://${HELM_REPO}/cni \
--namespace istio-system \
--kube-context ${CLUSTER2} \
--version ${ISTIO_IMAGE} \
-f - <<EOF
# Assigns IP addresses to multicluster services
ambient:
  dnsCapture: true
excludeNamespaces:
  - istio-system
  - kube-system
global:
  hub: ${REPO}
  tag: ${ISTIO_IMAGE}
  variant: distroless
  # platform: gke # UNCOMMENT FOR GKE
profile: ambient
EOF
```


Install ztunnel on cluster1 and cluster2
```bash
helm upgrade --install ztunnel oci://${HELM_REPO}/ztunnel \
--namespace istio-system \
--kube-context ${CLUSTER1} \
--version ${ISTIO_IMAGE} \
-f - <<EOF
configValidation: true
enabled: true
env:
  L7_ENABLED: "true"
  # Required when a unique trust domain is set for each cluster
  SKIP_VALIDATE_TRUST_DOMAIN: "true"
hub: ${REPO}
multiCluster:
  clusterName: cluster1
tag: ${ISTIO_IMAGE}
istioNamespace: istio-system
namespace: istio-system
network: cluster1
profile: ambient
proxy:
  clusterDomain: cluster.local
terminationGracePeriodSeconds: 29
variant: distroless
EOF
```

```bash
helm upgrade --install ztunnel oci://${HELM_REPO}/ztunnel \
--namespace istio-system \
--kube-context ${CLUSTER2} \
--version ${ISTIO_IMAGE} \
-f - <<EOF
configValidation: true
enabled: true
env:
  L7_ENABLED: "true"
  # Required when a unique trust domain is set for each cluster
  SKIP_VALIDATE_TRUST_DOMAIN: "true"
hub: ${REPO}
multiCluster:
  clusterName: cluster2
tag: ${ISTIO_IMAGE}
istioNamespace: istio-system
namespace: istio-system
network: cluster2
profile: ambient
proxy:
  clusterDomain: cluster.local
terminationGracePeriodSeconds: 29
variant: distroless
EOF
```

```bash
kubectl label namespace istio-system --context ${CLUSTER1} topology.istio.io/network=cluster1
kubectl label namespace istio-system --context ${CLUSTER2} topology.istio.io/network=cluster2
```


### Peer the clusters together

Expose using an east-west gateway:
```bash
istioctl --context=${CLUSTER1} multicluster expose --wait -n istio-gateways
istioctl --context=${CLUSTER2} multicluster expose --wait -n istio-gateways
```
Link clusters together:
```bash
istioctl multicluster link --contexts=$CLUSTER1,$CLUSTER2 -n istio-gateways
```


### Enable Istio for bookinfo Namespace

```bash
for context in ${CLUSTER1} ${CLUSTER2}; do
  kubectl --context ${context} label namespace bookinfo istio.io/dataplane-mode=ambient
done
```

Enable productpage to be multi-cluster on both clusters
```bash
for context in ${CLUSTER1} ${CLUSTER2}; do
  kubectl --context ${context}  -n bookinfo label service productpage solo.io/service-scope=global
  kubectl --context ${context}  -n bookinfo annotate service productpage  networking.istio.io/traffic-distribution=Any
done
```

### Expose Productpage using Istio Gateway

Apply the following Kubernetes Gateway API resources to cluster1 to expose productpage service using an Istio gateway:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: bookinfo-gateway
  namespace: bookinfo
spec:
  gatewayClassName: istio
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bookinfo
  namespace: bookinfo
spec:
  parentRefs:
  - name: bookinfo-gateway
  rules:
  - matches:
    - path:
        type: Exact
        value: /productpage
    - path:
        type: PathPrefix
        value: /static
    - path:
        type: Exact
        value: /login
    - path:
        type: Exact
        value: /logout
    - path:
        type: PathPrefix
        value: /api/v1/products
    # backendRefs:
    # - name: productpage
    #   port: 9080
    backendRefs:
    - kind: Hostname
      group: networking.istio.io
      name: productpage.bookinfo.mesh.internal
      port: 9080

```

Wait until a LB IP gets assigned to bookinfo-gateway-istio svc and then visit the app!

```bash
curl $(kubectl get svc -n bookinfo bookinfo-gateway-istio --context $CLUSTER1 -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")/productpage
```
Voila! This should be round robinning between productpage on both clusters.


### Istio Waypoints for L7 Functionality

Istio Waypoints enable Layer 7 traffic management in an Ambient Mesh, providing advanced capabilities like routing, authorization, observability, and security policies. Acting as dedicated traffic proxies, Waypoints handle HTTP, gRPC, and other application-layer protocols, seamlessly integrating with Istio’s security model to enforce fine-grained traffic control.

Let’s apply a Waypoint for the bookinfo namespace and create a header-based routing policy:
	•	Traffic going to reviews Service should route to reviews-v1 by default.
	•	Requests with the header end-user: jason should be directed to reviews-v2 instead.


```bash
for context in ${CLUSTER1} ${CLUSTER2}; do
  istioctl --context=${context} waypoint apply -n bookinfo
  kubectl --context=${context} label ns bookinfo istio.io/use-waypoint=waypoint
  kubectl --context=${context} apply -f ./reviews-v1.yaml 
done
```

# Egress Gateway

In addition to managing traffic coming into the mesh and within the mesh, ambient mesh can also manage traffic leaving the mesh. This includes observing the traffic and enforcing policies against it.

Just as a waypoint can be used for traffic addressed to a service inside your cluster, a gateway can be used for traffic that leaves your cluster.

In Istio, you can direct traffic to this gateway on a host-by-host basis using the ServiceEntry resource, which is bound to a waypoint used for egress control.

This section will only use $CLUSTER1.

First, we'll deploy an egress gateway in the `istio-egress` namespace, and call it `egress-gateway`

```bash
kubectl create namespace istio-egress
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: egress-gateway
  namespace: istio-egress
spec:
  gatewayClassName: istio-waypoint
  listeners:
  - name: mesh
    port: 15008
    protocol: HBONE
    allowedRoutes:
      namespaces:
        from: All
EOF
```

If you plan on creating SE's in the istio-egress namespace, you can label just the ns and not need to label every SE:
```
kubectl label ns istio-egress istio.io/use-waypoint=egress-gateway
```

Define httpbin.org on port 40 and 443 as external hosts using ServiceEntries in the `bookinfo` namespace. Notice that we're labeling the ServiceEntry to use the egress gateway

```yaml
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: httpbin.org
  namespace: bookinfo
  labels:
    istio.io/use-waypoint: egress-gateway
    istio.io/use-waypoint-namespace: istio-egress
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 80
    name: http
    protocol: HTTP
    targetPort: 443
  resolution: DNS
---
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: httpbin.org-tls
  namespace: bookinfo
spec:
  host: httpbin.org
  trafficPolicy:
    tls:
      mode: SIMPLE
EOF
```

Only allow ratings to call httpbin.org
```yaml
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: ratings-to-httpbin
  namespace: bookinfo
spec:
  targetRefs:
  - kind: ServiceEntry
    group: networking.istio.io
    name: httpbin.org
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/bookinfo/sa/bookinfo-ratings"]
EOF
```

You should now be able to call httpbin.org from ratings:
```bash
kubectl exec -it $(kubectl get pod -l app=ratings -n bookinfo -o jsonpath='{.items[0].metadata.name}') -n bookinfo -- curl -s httpbin.org/get
```

But NOT reviews:
```
kubectl exec -it $(kubectl get pod -l app=reviews -n bookinfo -o jsonpath='{.items[0].metadata.name}') -n bookinfo -- curl -s httpbin.org/get
```

### Block unmatched traffic
But what about hosts that dosn't have a matching ServiceEntry. For example, if we try `jsonplaceholder.typicode.com/todos/1`, it works:

```bash
kubectl exec -it $(kubectl get pod -l app=reviews -n bookinfo -o jsonpath='{.items[0].metadata.name}') -n bookinfo -- curl -s http://jsonplaceholder.typicode.com/todos/1
```

To block this, we need [egress policies](https://ambientmesh.io/docs/traffic/mesh-egress/#policy-enforcement), which instruct ztunnel how to manage unmatched traffic.

We need to update ztunnel with our policy:
```bash
helm upgrade --install ztunnel oci://${HELM_REPO}/ztunnel \
--namespace istio-system \
--kube-context ${CLUSTER1} \
--version ${ISTIO_IMAGE} \
-f - <<EOF
configValidation: true
enabled: true
env:
  L7_ENABLED: "true"
  # Required when a unique trust domain is set for each cluster
  SKIP_VALIDATE_TRUST_DOMAIN: "true"
hub: ${REPO}
multiCluster:
  clusterName: cluster1
tag: ${ISTIO_IMAGE}
istioNamespace: istio-system
namespace: istio-system
network: cluster1
profile: ambient
proxy:
  clusterDomain: cluster.local
terminationGracePeriodSeconds: 29
variant: distroless
egressPolicies:
# Allow the egress gateway to send anywhere, so we do not loop
- namespaces: [istio-egress]
  policy: Passthrough
# For anything else (that doesn't have a SE), Deny
- policy: Deny
  matchCidrs:
  - 0.0.0.0/0
  - ::/0
EOF
```
Let's try again:
```bash
kubectl exec -it $(kubectl get pod -l app=reviews -n bookinfo -o jsonpath='{.items[0].metadata.name}') -n bookinfo -- curl -s http://jsonplaceholder.typicode.com/todos/1
```
You should see something similar to: `command terminated with exit code 56`

## Gloo Management Plane

Optionally, you can deploy the Gloo Management Plane that provides many benefits and features. For this lab, we'll just focus on the UI and the service graph. 

Cluster1 will act as the management cluster and workload cluster:
```bash
helm repo add gloo-platform https://storage.googleapis.com/gloo-platform/helm-charts
helm repo update

helm upgrade --kube-context ${CLUSTER1} -i gloo-platform-crds gloo-platform/gloo-platform-crds -n gloo-mesh --create-namespace --version=2.7.1
helm upgrade --kube-context ${CLUSTER1} -i gloo-platform gloo-platform/gloo-platform -n gloo-mesh --version 2.7.1 --values mgmt-values.yaml \
  --set licensing.glooMeshLicenseKey=$GLOO_MESH_LICENSE_KEY
```

Then, you need to set the environment variables with the management plane addresses. These variables will be used to configure the Gloo Mesh agents in cluster2:

```bash
export ENDPOINT_GLOO_MESH=$(kubectl --context ${CLUSTER1} -n gloo-mesh get svc gloo-mesh-mgmt-server -o jsonpath='{.status.loadBalancer.ingress[0].ip}{.status.loadBalancer.ingress[0].hostname}'):9900
export HOST_GLOO_MESH=$(echo ${ENDPOINT_GLOO_MESH%:*})
export ENDPOINT_TELEMETRY_GATEWAY=$(kubectl --context ${CLUSTER1} -n gloo-mesh get svc gloo-telemetry-gateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}{.status.loadBalancer.ingress[0].hostname}'):4317
export ENDPOINT_GLOO_MESH_UI=$(kubectl --context ${CLUSTER1} -n gloo-mesh get svc gloo-mesh-ui -o jsonpath='{.status.loadBalancer.ingress[0].ip}{.status.loadBalancer.ingress[0].hostname}'):8090
```
Check that the variables have correct values:
```
echo $HOST_GLOO_MESH
echo $ENDPOINT_GLOO_MESH
```

Tell the management plane that cluster2 will connect to it
```bash
kubectl apply --context ${CLUSTER1} -f - <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: KubernetesCluster
metadata:
  name: cluster2
  namespace: gloo-mesh
spec:
  clusterDomain: cluster.local
EOF
```

Copy the root and token over to cluster2. This will be used by the agent to perform mtls connection while connecting to the mgmt plane.
```bash
kubectl --context ${CLUSTER2} create ns gloo-mesh

kubectl get secret relay-root-tls-secret -n gloo-mesh --context ${CLUSTER1} -o jsonpath='{.data.ca\.crt}' | base64 -d > ca.crt
kubectl create secret generic relay-root-tls-secret -n gloo-mesh --context ${CLUSTER2} --from-file ca.crt=ca.crt
rm ca.crt

kubectl get secret relay-identity-token-secret -n gloo-mesh --context ${CLUSTER1} -o jsonpath='{.data.token}' | base64 -d > token
kubectl create secret generic relay-identity-token-secret -n gloo-mesh --context ${CLUSTER2} --from-file token=token
rm token
```

Deploy the agent in cluster2
```bash
helm upgrade --install gloo-platform-crds gloo-platform-crds \
  --repo https://storage.googleapis.com/gloo-platform/helm-charts \
  --namespace gloo-mesh \
  --set installEnterpriseCrds=false \
  --kube-context ${CLUSTER2} \
  --version 2.7.1

helm upgrade --install gloo-platform gloo-platform \
  --repo https://storage.googleapis.com/gloo-platform/helm-charts \
  --namespace gloo-mesh \
  --kube-context ${CLUSTER2} \
  --version 2.7.1 \
  -f -<<EOF
common:
  cluster: cluster2
glooAgent:
  enabled: true
  relay:
    serverAddress: "${ENDPOINT_GLOO_MESH}"
    authority: gloo-mesh-mgmt-server.gloo-mesh
telemetryCollector:
  enabled: true
  config:
    exporters:
      otlp:
        endpoint: "${ENDPOINT_TELEMETRY_GATEWAY}"
glooAnalyzer:
  enabled: true
EOF
```

Launch the UI:
```
kubectl -n gloo-mesh port-forward deployment/gloo-mesh-ui 8090
```

localhost:8090:
![Gloo Mesh UI](gloo-mesh-ui.png)
