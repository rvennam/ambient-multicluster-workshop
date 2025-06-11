# Solo.io Istio Ambient Single-cluster Workshop

In this workshop, you will set up Istio Ambient in a single cluster environment, deploy a sample Bookinfo app, and explore how Solo.io Ambient enables secure service-to-service communication in all directions.

### Env

1. Create a cluster and set the env vars below to their context
```bash
export CLUSTER1=gke_ambient_one # UPDATE THIS
export GLOO_MESH_LICENSE_KEY=<update>  # UPDATE THIS
```
2. Download Solo's 1.25.3 `istioctl` Binary:
```bash
OS=$(uname | tr '[:upper:]' '[:lower:]' | sed -E 's/darwin/osx/')
ARCH=$(uname -m | sed -E 's/aarch/arm/; s/x86_64/amd64/; s/armv7l/armv7/')

mkdir -p ~/.istioctl/bin
curl -sSL https://storage.googleapis.com/istio-binaries-e038d180f90a/1.25.3-solo/istioctl-1.25.3-solo-${OS}-${ARCH}.tar.gz | tar xzf - -C ~/.istioctl/bin
chmod +x ~/.istioctl/bin/istioctl

export PATH=${HOME}/.istioctl/bin:${PATH}
```
3. Verify using `istioctl version`


### Deploy Bookinfo sample to cluster
```bash
  kubectl --context $CLUSTER1 create ns bookinfo 
  kubectl --context $CLUSTER1 apply -n bookinfo -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/bookinfo/platform/kube/bookinfo.yaml
  kubectl --context $CLUSTER1 apply -n bookinfo -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/bookinfo/platform/kube/bookinfo-versions.yaml
```

### Install Istio using Gloo Operator

To use HELM instead, see [Helm Instructions](./README_helm.md)

Install the operator
```bash
  helm upgrade -i --kube-context=${CLUSTER1} gloo-operator \
    oci://us-docker.pkg.dev/solo-public/gloo-operator-helm/gloo-operator \
    --version 0.2.3 -n gloo-system --create-namespace \
    --set manager.env.SOLO_ISTIO_LICENSE_KEY=${GLOO_MESH_LICENSE_KEY} \
    --set manager.image.repository=us-docker.pkg.dev/solo-public/gloo-operator/gloo-operator &
```
Use the `ServiceMeshController` resource to install Istio on both clusters

```bash
kubectl --context=${CLUSTER1} apply -f - <<EOF
apiVersion: operator.gloo.solo.io/v1
kind: ServiceMeshController
metadata:
  name: istio
spec:
  version: 1.25.3
  cluster: cluster1
  network: cluster1
EOF
```

### 
### Enable Istio for bookinfo Namespace

```bash
  kubectl --context ${CLUSTER1} label namespace bookinfo istio.io/dataplane-mode=ambient
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
    - name: productpage
      port: 9080
EOF
```

Wait until a LB IP gets assigned to bookinfo-gateway-istio svc and then visit the app!

```bash
curl $(kubectl get svc -n bookinfo bookinfo-gateway-istio --context $CLUSTER1 -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")/productpage
```
Voila! You now have an route to a fully mtls'd environment

### Istio Waypoints for L7 Functionality

Istio Waypoints enable Layer 7 traffic management in an Ambient Mesh, providing advanced capabilities like routing, authorization, observability, and security policies. Acting as dedicated traffic proxies, Waypoints handle HTTP, gRPC, and other application-layer protocols, seamlessly integrating with Istio’s security model to enforce fine-grained traffic control.

Let’s apply a Waypoint for the bookinfo namespace and create a header-based routing policy:
	•	Traffic going to reviews Service should route to reviews-v1 by default.
	•	Requests with the header end-user: jason should be directed to reviews-v2 instead.


```bash
  istioctl --context=${CLUSTER1} waypoint apply -n bookinfo
  kubectl --context=${CLUSTER1} label ns bookinfo istio.io/use-waypoint=waypoint
  kubectl --context=${CLUSTER1} apply -f ./reviews-v1.yaml 
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

### Gloo Management Plane

Optionally, you can deploy the Gloo Management Plane that provides many benefits and features. For this lab, we'll just focus on the UI and the service graph. 

Start by downloading the meshctl cli
```
curl -sL https://run.solo.io/meshctl/install | GLOO_MESH_VERSION=v2.7.1 sh -
export PATH=$HOME/.gloo-mesh/bin:$PATH
```


Cluster1 will act as the management cluster and workload cluster: (see [mgmt-values.yaml](./mgmt-values.yaml) for reference)
```bash
helm repo add gloo-platform https://storage.googleapis.com/gloo-platform/helm-charts
helm repo update

helm upgrade -i gloo-platform-crds gloo-platform/gloo-platform-crds -n gloo-mesh --create-namespace --version=2.7.1 --kube-context=$CLUSTER1
helm upgrade -i gloo-platform gloo-platform/gloo-platform -n gloo-mesh --version 2.7.1 --kube-context=$CLUSTER1 --values mgmt-values.yaml \
  --set licensing.glooMeshLicenseKey=$GLOO_MESH_LICENSE_KEY
```

Launch the UI:
```
meshctl dashboard
```
![Gloo Mesh UI](gloo-mesh-ui.png)
