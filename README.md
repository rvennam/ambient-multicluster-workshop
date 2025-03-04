# Solo.io Istio Ambient Multi-cluster Workshop

In this workshop, you will set up Istio Ambient in a multi-cluster environment, deploy a sample Bookinfo app, and explore how Solo.io Ambient enables secure service-to-service communication in all directions.

![overview](overview.png)

|[![Gloo Mesh Ambient Multi-Cluster Video](yt-video.png)](https://youtu.be/18dpBukOYSk) |
|:-----------------------------:|
### Env

1. Create two clusters and set the env vars below to their context
2. Install Solo's IstioCTL Binary:
```bash
# URL format is as follows: curl -fSsLO https://storage.googleapis.com/istio-binaries-${OBFUSCATED_STRING}/${VERSION}-${FLAVOR}/istio-${VERSION}-${FLAVOR}-${OS}-${ARCH}.tar.gz
# For example 1.24.3 m1 solo istioctl download: curl -fSsLO https://storage.googleapis.com/istio-binaries-4d37697f9711/1.24.3-solo/istio-1.24.3-solo-osx-arm64.tar.gz
OS=$(uname | tr '[:upper:]' '[:lower:]' | sed -E 's/darwin/osx/')
ARCH=$(uname -m | sed -E 's/aarch/arm/; s/x86_64/amd64/; s/armv7l/armv7/')

mkdir -p ~/.istioctl/bin
curl -sSL https://storage.googleapis.com/istio-binaries-e038d180f90a/1.25.0-solo/istioctl-1.25.0-solo-${OS}-${ARCH}.tar.gz | tar xzf - -C ~/.istioctl/bin
chmod +x ~/.istioctl/bin/istioctl

export PATH=${HOME}/.istioctl/bin:${PATH}
```

4. Download the latest istio-solo, 1.24.3 or greater. Verify using `istioctl version`

Set env vars
```bash
export CLUSTER1=gke_ambient_one
export CLUSTER2=gke_ambient_two
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

### Install Istio on both clusters using Gloo Operator

Install the operator
```bash
for context in ${CLUSTER1} ${CLUSTER2}; do
  helm upgrade --install --kube-context=${context} gloo-operator oci://us-docker.pkg.dev/solo-public/gloo-operator-helm/gloo-operator --version 0.1.0-rc.1 -n gloo-system --create-namespace &
done
```
Use the `ServiceMeshController` resource to install Istio on both clusters

```bash
kubectl --context=${CLUSTER1} apply -f - <<EOF
apiVersion: operator.gloo.solo.io/v1
kind: ServiceMeshController
metadata:
  name: istio
spec:
  version: 1.24.3
  cluster: cluster1
  network: cluster1
EOF

kubectl --context=${CLUSTER2} apply -f - <<EOF
apiVersion: operator.gloo.solo.io/v1
kind: ServiceMeshController
metadata:
  name: istio
spec:
  version: 1.24.3
  cluster: cluster2
  network: cluster2
EOF
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

## Gloo Management Plane

Optionally, you can deploy the Gloo Management Plane that provides many benefits and features. For this lab, we'll just focus on the UI and the service graph. 

Start by downloading the meshctl cli
```
curl -sL https://run.solo.io/meshctl/install | GLOO_MESH_VERSION=v2.7.0 sh -
export PATH=$HOME/.gloo-mesh/bin:$PATH
```


Cluster1 will act as the management cluster and workload cluster:
```bash
helm repo add gloo-platform https://storage.googleapis.com/gloo-platform/helm-charts
helm repo update

helm upgrade -i gloo-platform-crds gloo-platform/gloo-platform-crds -n gloo-mesh --create-namespace --version=2.7.0
helm upgrade -i gloo-platform gloo-platform/gloo-platform -n gloo-mesh --version 2.7.0 --values mgmt-values.yaml \
  --set licensing.glooMeshLicenseKey=$GLOO_MESH_LICENSE_KEY
```

Then, register cluster2 as a workload cluster to cluster1:
```bash
export TELEMETRY_GATEWAY_ADDRESS=$(kubectl get svc -n gloo-mesh gloo-telemetry-gateway --context $CLUSTER1 -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}"):4317

meshctl cluster register cluster2  --kubecontext $CLUSTER1 --profiles gloo-core-agent --remote-context $CLUSTER2 --telemetry-server-address $TELEMETRY_GATEWAY_ADDRESS
```

Launch the UI:
```
meshctl dashboard
```
![Gloo Mesh UI](gloo-mesh-ui.png)
