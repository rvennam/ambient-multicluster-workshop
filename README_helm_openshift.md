# Solo.io Istio Ambient Multi-cluster Workshop - OpenShift Edition

In this workshop, you will set up Istio Ambient in a multi-cluster OpenShift environment, deploy a sample Bookinfo app, and explore how Solo.io Ambient enables secure service-to-service communication in all directions.

![overview](overview.png)

|[![Gloo Mesh Ambient Multi-Cluster Video](yt-video.png)](https://youtu.be/18dpBukOYSk) |
|:-----------------------------:|

## OpenShift-Specific Notes

This guide is specifically tailored for OpenShift deployments and includes the following OpenShift-specific configurations:
- Multus CNI provider configuration
- SELinux context configuration (`spc_t`)
- Platform-specific settings for OpenShift

### Env

1. Create two OpenShift clusters and set the env vars below to their context

```bash
export CLUSTER1=openshift_cluster_one # UPDATE THIS
export CLUSTER2=openshift_cluster_two # UPDATE THIS
export GLOO_MESH_LICENSE_KEY=<update>  # UPDATE THIS

export ISTIO_VERSION=1.28.3-patch0
export REPO_KEY=594e990587b9
export ISTIO_IMAGE=${ISTIO_VERSION}-solo
export REPO=us-docker.pkg.dev/gloo-mesh/istio-${REPO_KEY}
export HELM_REPO=us-docker.pkg.dev/gloo-mesh/istio-helm-${REPO_KEY}
```

2. Download Solo's `istioctl` Binary:
```bash
OS=$(uname | tr '[:upper:]' '[:lower:]' | sed -E 's/darwin/osx/')
ARCH=$(uname -m | sed -E 's/aarch/arm/; s/x86_64/amd64/; s/armv7l/armv7/')

mkdir -p ~/.istioctl/bin
curl -sSL https://storage.googleapis.com/istio-binaries-${REPO_KEY}/${ISTIO_VERSION}-solo/istioctl-${ISTIO_VERSION}-solo-${OS}-${ARCH}.tar.gz | tar xzf - -C ~/.istioctl/bin
chmod +x ~/.istioctl/bin/istioctl

export PATH=${HOME}/.istioctl/bin:${PATH}
```

3. Verify using `istioctl version`

### Clean up previous Istio installations

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
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.28.3 sh -
cd istio-1.28.3

mkdir -p certs
pushd certs
make -f ../tools/certs/Makefile.selfsigned.mk root-ca

function create_cacerts_secret() {
  context=${1:?context}
  cluster=${2:?cluster}
  make -f ../tools/certs/Makefile.selfsigned.mk ${cluster}-cacerts
  kubectl --context=${context} create ns istio-system || true
  kubectl --context=${context} create secret generic cacerts -n istio-system \
    --from-file=${cluster}/ca-cert.pem \
    --from-file=${cluster}/ca-key.pem \
    --from-file=${cluster}/root-cert.pem \
    --from-file=${cluster}/cert-chain.pem
}

create_cacerts_secret ${CLUSTER1} cluster1
create_cacerts_secret ${CLUSTER2} cluster2

cd ../..
```

Install Istio CRD's on cluster1 and cluster2:

```bash
helm upgrade --install istio-base oci://${HELM_REPO}/base \
--namespace istio-system \
--kube-context ${CLUSTER1} \
--create-namespace \
--version ${ISTIO_IMAGE} \
-f - <<EOF
defaultRevision: ""
profile: ambient
# OpenShift platform specification
global:
  platform: openshift
EOF
```

```bash
helm upgrade --install istio-base oci://${HELM_REPO}/base \
--namespace istio-system \
--kube-context ${CLUSTER2} \
--create-namespace \
--version ${ISTIO_IMAGE} \
-f - <<EOF
defaultRevision: ""
profile: ambient
# OpenShift platform specification
global:
  platform: openshift
EOF
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
  # PILOT_ENABLE_K8S_SELECT_WORKLOAD_ENTRIES: "false"
  # Required when meshConfig.trustDomain is set
  PILOT_SKIP_VALIDATE_TRUST_DOMAIN: "true"
global:
  hub: ${REPO}
  multiCluster:
    clusterName: cluster1
  network: cluster1
  # OpenShift platform specification
  platform: openshift
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
  # OpenShift platform specification
  platform: openshift
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

Install istio-cni on cluster1 and cluster2. **NOTE**: For OpenShift, use Multus provider configuration.

```bash
helm upgrade --install istio-cni oci://${HELM_REPO}/cni \
--namespace istio-system \
--kube-context ${CLUSTER1} \
--version ${ISTIO_IMAGE} \
-f - <<EOF
ambient:
  dnsCapture: true
# OpenShift-specific CNI configuration
cni:
  cniBinDir: /var/lib/cni/bin
  cniConfDir: /etc/cni/multus/net.d
  chained: false
  cniConfFileName: "istio-cni.conf"
  provider: "multus"
excludeNamespaces:
  - istio-system
  - kube-system
global:
  hub: ${REPO}
  tag: ${ISTIO_IMAGE}
  variant: distroless
  # OpenShift platform specification
  platform: openshift
# OpenShift SELinux configuration
seLinuxOptions:
  type: spc_t
profile: ambient
EOF
```

```bash
helm upgrade --install istio-cni oci://${HELM_REPO}/cni \
--namespace istio-system \
--kube-context ${CLUSTER2} \
--version ${ISTIO_IMAGE} \
-f - <<EOF
ambient:
  dnsCapture: true
# OpenShift-specific CNI configuration
cni:
  cniBinDir: /var/lib/cni/bin
  cniConfDir: /etc/cni/multus/net.d
  chained: false
  cniConfFileName: "istio-cni.conf"
  provider: "multus"
excludeNamespaces:
  - istio-system
  - kube-system
global:
  hub: ${REPO}
  tag: ${ISTIO_IMAGE}
  variant: distroless
  # OpenShift platform specification
  platform: openshift
# OpenShift SELinux configuration
seLinuxOptions:
  type: spc_t
profile: ambient
EOF
```

Install ztunnel on cluster1 and cluster2.

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

**Create an east-west gateway:**

```bash
istioctl --context=${CLUSTER1} multicluster expose -n istio-gateways
istioctl --context=${CLUSTER2} multicluster expose -n istio-gateways
```

<details>

<summary>Instead of using istioctl, you can also apply yaml</summary>

```bash
kubectl apply --context $CLUSTER1 -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  labels:
    istio.io/expose-istiod: "15012"
    topology.istio.io/network: cluster1
  name: istio-eastwest
  namespace: istio-gateways
spec:
  gatewayClassName: istio-eastwest
  listeners:
  - name: cross-network
    port: 15008
    protocol: HBONE
    tls:
      mode: Passthrough
  - name: xds-tls
    port: 15012
    protocol: TLS
    tls:
      mode: Passthrough
EOF
```

```bash
kubectl apply --context $CLUSTER2 -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  labels:
    istio.io/expose-istiod: "15012"
    topology.istio.io/network: cluster2
  name: istio-eastwest
  namespace: istio-gateways
spec:
  gatewayClassName: istio-eastwest
  listeners:
  - name: cross-network
    port: 15008
    protocol: HBONE
    tls:
      mode: Passthrough
  - name: xds-tls
    port: 15012
    protocol: TLS
    tls:
      mode: Passthrough
EOF
```
</details>

**Link clusters together:**

```bash
istioctl multicluster link --contexts=$CLUSTER1,$CLUSTER2 -n istio-gateways
```

<details>

<summary>Instead of using istioctl, you can also apply yaml</summary>

```bash
export CLUSTER1_EW_ADDRESS=$(kubectl get svc -n istio-gateways istio-eastwest --context $CLUSTER1 -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")
export CLUSTER2_EW_ADDRESS=$(kubectl get svc -n istio-gateways istio-eastwest --context $CLUSTER2 -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")

echo "Cluster 1 east-west gateway: $CLUSTER1_EW_ADDRESS"
echo "Cluster 2 east-west gateway: $CLUSTER2_EW_ADDRESS"

kubectl apply --context $CLUSTER1 -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  annotations:
    gateway.istio.io/service-account: istio-eastwest
    gateway.istio.io/trust-domain: cluster2
  labels:
    topology.istio.io/network: cluster2
  name: istio-remote-peer-cluster2
  namespace: istio-gateways
spec:
  addresses:
  - type: IPAddress
    value: $CLUSTER2_EW_ADDRESS
  gatewayClassName: istio-remote
  listeners:
  - name: cross-network
    port: 15008
    protocol: HBONE
    tls:
      mode: Passthrough
  - name: xds-tls
    port: 15012
    protocol: TLS
    tls:
      mode: Passthrough
EOF

kubectl apply --context $CLUSTER2 -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  annotations:
    gateway.istio.io/service-account: istio-eastwest
    gateway.istio.io/trust-domain: cluster1
  labels:
    topology.istio.io/network: cluster1
  name: istio-remote-peer-cluster1
  namespace: istio-gateways
spec:
  addresses:
  - type: IPAddress
    value: $CLUSTER1_EW_ADDRESS
  gatewayClassName: istio-remote
  listeners:
  - name: cross-network
    port: 15008
    protocol: HBONE
    tls:
      mode: Passthrough
  - name: xds-tls
    port: 15012
    protocol: TLS
    tls:
      mode: Passthrough
EOF
```
</details>

**Let's make sure we did everything right!**

```bash
istioctl --context $CLUSTER1 multicluster check
istioctl --context $CLUSTER2 multicluster check
```

We should expect to see:
```
✅ License Check: license is valid for multicluster
✅ Pod Check (istiod): all pods healthy
✅ Pod Check (ztunnel): all pods healthy
✅ Pod Check (eastwest gateway): all pods healthy
✅ Gateway Check: all eastwest gateways programmed
✅ Peers Check: all clusters connected
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

Istio Waypoints enable Layer 7 traffic management in an Ambient Mesh, providing advanced capabilities like routing, authorization, observability, and security policies. Acting as dedicated traffic proxies, Waypoints handle HTTP, gRPC, and other application-layer protocols, seamlessly integrating with Istio's security model to enforce fine-grained traffic control.

Let's apply a Waypoint for the bookinfo namespace and create a header-based routing policy:
	•	Traffic going to reviews Service should route to reviews-v1 by default.
	•	Requests with the header end-user: jason should be directed to reviews-v2 instead.


```bash
for context in ${CLUSTER1} ${CLUSTER2}; do
  istioctl --context=${context} waypoint apply -n bookinfo
  kubectl --context=${context} label ns bookinfo istio.io/use-waypoint=waypoint
  kubectl --context=${context} apply -f ./reviews-v1.yaml
done
```

