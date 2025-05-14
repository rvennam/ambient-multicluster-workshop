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
```
2. Download Solo's 1.25.0 `istioctl` Binary:
```bash
OS=$(uname | tr '[:upper:]' '[:lower:]' | sed -E 's/darwin/osx/')
ARCH=$(uname -m | sed -E 's/aarch/arm/; s/x86_64/amd64/; s/armv7l/armv7/')

mkdir -p ~/.istioctl/bin
curl -sSL https://storage.googleapis.com/istio-binaries-e038d180f90a/1.25.0-solo/istioctl-1.25.0-solo-${OS}-${ARCH}.tar.gz | tar xzf - -C ~/.istioctl/bin
chmod +x ~/.istioctl/bin/istioctl

export PATH=${HOME}/.istioctl/bin:${PATH}
```
3. Verify using `istioctl version`


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
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.25.1 sh -
cd istio-1.25.1
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
```

### Install Istio on both clusters using Gloo Operator

To use HELM instead, see [Helm Instructions](./README_helm.md)

Install the operator
```bash
for context in ${CLUSTER1} ${CLUSTER2}; do
  helm upgrade -i --kube-context=${context} gloo-operator \
    oci://us-docker.pkg.dev/solo-public/gloo-operator-helm/gloo-operator \
    --version 0.2.3 -n gloo-system --create-namespace \
    --set manager.env.SOLO_ISTIO_LICENSE_KEY=${GLOO_MESH_LICENSE_KEY} \
    --set manager.image.repository=us-docker.pkg.dev/solo-public/gloo-operator/gloo-operator &
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
  version: 1.25.1
  cluster: cluster1
  network: cluster1
EOF

kubectl --context=${CLUSTER2} apply -f - <<EOF
apiVersion: operator.gloo.solo.io/v1
kind: ServiceMeshController
metadata:
  name: istio
spec:
  version: 1.25.1
  cluster: cluster2
  network: cluster2
EOF
```

### Peer the clusters together

Expose using an east-west gateway:
```bash
kubectl create ns istio-gateways --context ${CLUSTER1}
kubectl create ns istio-gateways --context ${CLUSTER2}
```

```
istioctl --context=${CLUSTER1} multicluster expose --wait -n istio-gateways
istioctl --context=${CLUSTER2} multicluster expose --wait -n istio-gateways
```
<details>

<summary>Instead of using istioctl, you can also apply yaml</summary>
```
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
```
```
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
```
</details>


**Link clusters together:**

```bash
istioctl multicluster link --contexts=$CLUSTER1,$CLUSTER2 -n istio-gateways
```

<details>

<summary>Instead of using istioctl, you can also apply yaml</summary>

```bash
export CLUSTER1_EW_ADDRESS=$(kubectl get svc -n cnp-istio istio-eastwest --context $CLUSTER1 -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")
export CLUSTER2_EW_ADDRESS=$(kubectl get svc -n cnp-istio istio-eastwest --context $CLUSTER2 -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")

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
  namespace: cnp-istio
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
  namespace: cnp-istio
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

Istio multi-cluster is installed and ready to go!

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
kubectl --context=${CLUSTER1} apply -f - <<EOF
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
EOF
```

Wait until a LB IP gets assigned to bookinfo-gateway-istio svc and then visit the app!

```bash
curl $(kubectl get svc -n bookinfo bookinfo-gateway-istio --context $CLUSTER1 -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")/productpage
```
Voila! This should be round robinning between productpage on both clusters.


### Automatic and Manual Failover

**productpage-v1 failover**
Scale down productpage on cluster1 to simulate a failure:
```bash
kubectl scale deploy productpage-v1 --replicas=0 --context $CLUSTER1
```
Visit the application in your browser and you'll see traffic is not impacted because we're failing over to cluster2 automatically.

Revert the changes:
```bash
kubectl scale deploy productpage-v1 --replicas=1 --context $CLUSTER1
```
**details-v1 failover**
We can also scale down other services. Lets enable `details` to be multi-cluster and scale it down
```bash
kubectl --context $CLUSTER1 -n bookinfo label service details solo.io/service-scope=global-only 
kubectl --context $CLUSTER2  -n bookinfo label service details solo.io/service-scope=global-only 
```
```bash
kubectl scale deploy details-v1 --replicas=0 --context $CLUSTER1
```

Visit the application in your browser and you'll see traffic is not impacted because we're failing over from productpage.cluster1 to bookinfo.cluster2 automatically.


### Discovery and Migration of Microservices

To simulate migration of microservices between clusters, lets deploy httpbin service to Cluster1 and call it from ratings.

Deploy httpbin and mark it as global
```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/httpbin/httpbin.yaml -n bookinfo --context $CLUSTER1
kubectl -n bookinfo label service httpbin solo.io/service-scope=global --context $CLUSTER1
```
Test ratings-v1.cluster1 -> httpbin.cluster1
```bash
kubectl exec deploy/ratings-v1 -n bookinfo --context $CLUSTER1 -- curl httpbin.bookinfo.mesh.internal:8000/headers
```
Now, lets move it to cluster2
```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/httpbin/httpbin.yaml -n bookinfo --context $CLUSTER2
kubectl -n bookinfo label service httpbin solo.io/service-scope=global --context $CLUSTER2
kubectl delete -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/httpbin/httpbin.yaml -n bookinfo --context $CLUSTER1
```

Test ratings-v1.cluster1 -> httpbin.cluster2 with the same command as before:
```bash
kubectl exec deploy/ratings-v1 -n bookinfo --context $CLUSTER1 -- curl httpbin.bookinfo.mesh.internal:8000/headers
```

### Intelligent Traffic Distribution

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

### Gradual Migration to Istio Ambient Mesh

Create a ns called legacy which will have sidecar based services. Deploy sleep and httpbin sample apps to it
```bash
kubectl create ns legacy --context $CLUSTER1
kubectl label ns legacy istio-injection=enabled --context $CLUSTER1
kubectl apply -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/sleep/sleep.yaml -n legacy  --context $CLUSTER1
kubectl apply -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/httpbin/httpbin.yaml -n legacy --context $CLUSTER1
```
**Sidecar to Ambient**
From sleep, call local and global ambient services
```
kubectl exec deploy/sleep -n legacy -- curl productpage.bookinfo.mesh.internal:9080
kubectl exec deploy/sleep -n legacy -- curl productpage.bookinfo:9080
```
**Ambient to Sidecar**
From ratings.bookinfo, call httpbin
```
kubectl exec deploy/ratings-v1 -n bookinfo -- curl httpbin.legacy:8000
```

### Centralized Multicluster Management

Optionally, you can deploy the Gloo Management Plane that provides many benefits and features. For this lab, we'll just focus on the UI and the service graph. 

Start by downloading the meshctl cli
```
curl -sL https://run.solo.io/meshctl/install | GLOO_MESH_VERSION=v2.8.0 sh -
export PATH=$HOME/.gloo-mesh/bin:$PATH
```


Cluster1 will act as the management cluster and workload cluster: (see [mgmt-values.yaml](./mgmt-values.yaml) for reference)
```bash
helm repo add gloo-platform https://storage.googleapis.com/gloo-platform/helm-charts
helm repo update

helm upgrade -i gloo-platform-crds gloo-platform/gloo-platform-crds -n gloo-mesh --create-namespace --version=2.8.0 --kube-context=$CLUSTER1
helm upgrade -i gloo-platform gloo-platform/gloo-platform -n gloo-mesh --version 2.8.0 --kube-context=$CLUSTER1 --values mgmt-values.yaml \
  --set licensing.glooMeshLicenseKey=$GLOO_MESH_LICENSE_KEY
```

Then, register cluster2 as a workload cluster to cluster1:
```bash
export TELEMETRY_GATEWAY_ADDRESS=$(kubectl get svc -n gloo-mesh gloo-telemetry-gateway --context $CLUSTER1 -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}"):4317

meshctl cluster register cluster2  --kubecontext $CLUSTER1 --profiles gloo-mesh-agent --remote-context $CLUSTER2 --telemetry-server-address $TELEMETRY_GATEWAY_ADDRESS
```

Launch the UI:
```
meshctl dashboard
```
![Gloo Mesh UI](gloo-mesh-ui.png)
