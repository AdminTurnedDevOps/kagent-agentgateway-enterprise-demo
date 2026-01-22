# Single Cluster Install

## Env Vars
```
export AGENTGATEWAY_LICENSE_KEY=

export ANTHROPIC_API_KEY=
```

## Ambient Config
```
export ISTIO_VERSION=1.27.0
export ISTIO_IMAGE=${ISTIO_VERSION}-solo
export REPO_KEY=d11c80c0c3fc
export REPO=us-docker.pkg.dev/gloo-mesh/istio-${REPO_KEY}
export HELM_REPO=us-docker.pkg.dev/gloo-mesh/istio-helm-${REPO_KEY}
export SOLO_ISTIO_LICENSE_KEY=
```

```
OS=$(uname | tr '[:upper:]' '[:lower:]' | sed -E 's/darwin/osx/')
ARCH=$(uname -m | sed -E 's/aarch/arm/; s/x86_64/amd64/; s/armv7l/armv7/')

mkdir -p ~/.istioctl/bin
curl -sSL https://storage.googleapis.com/istio-binaries-$REPO_KEY/1.27.1-solo/istioctl-1.27.1-solo-$OS-$ARCH.tar.gz | tar xzf - -C ~/.istioctl/bin
chmod +x ~/.istioctl/bin/istioctl

export PATH=${HOME}/.istioctl/bin:${PATH}
istioctl version --remote=false
```

```
helm upgrade --install istio-base oci://${HELM_REPO}/base \
--namespace istio-system \
--create-namespace \
--version ${ISTIO_IMAGE} \
-f - <<EOF
defaultRevision: ""
profile: ambient
EOF
```

```
helm upgrade --install istiod oci://${HELM_REPO}/istiod \
--namespace istio-system \
--version ${ISTIO_IMAGE} \
-f - <<EOF
global:
  hub: ${REPO}
  proxy:
    clusterDomain: cluster.local
  tag: ${ISTIO_IMAGE}
meshConfig:
  accessLogFile: /dev/stdout
  defaultConfig:
    proxyMetadata:
      ISTIO_META_DNS_AUTO_ALLOCATE: "true"
      ISTIO_META_DNS_CAPTURE: "true"
env:
  PILOT_ENABLE_IP_AUTOALLOCATE: "true"
  PILOT_SKIP_VALIDATE_TRUST_DOMAIN: "true"
pilot:
  cni:
    namespace: istio-system
    enabled: true
profile: ambient
license:
  value: ${SOLO_LICENSE_KEY}
  # Uncomment if you prefer to specify your license secret
  # instead of an inline value.
  # secretRef:
  #   name:
  #   namespace:
EOF
```

```
helm upgrade --install istio-cni oci://${HELM_REPO}/cni \
--namespace istio-system \
--version ${ISTIO_IMAGE} \
-f - <<EOF
ambient:
  dnsCapture: true
excludeNamespaces:
  - istio-system
  - kube-system
global:
  hub: ${REPO}
  tag: ${ISTIO_IMAGE}
  platform: gke
profile: ambient
cni:
  priorityClassName: ""
EOF

# Patch the DaemonSet to remove priority class (workaround for chart not respecting the value) if on GKE
kubectl patch daemonset istio-cni-node -n istio-system -p '{"spec":{"template":{"spec":{"priorityClassName":null}}}}'
```

```
helm upgrade --install ztunnel oci://${HELM_REPO}/ztunnel \
--namespace istio-system \
--version ${ISTIO_IMAGE} \
-f - <<EOF
configValidation: true
enabled: true
env:
  L7_ENABLED: "true"
hub: ${REPO}
istioNamespace: istio-system
namespace: istio-system
profile: ambient
proxy:
  clusterDomain: cluster.local
tag: ${ISTIO_IMAGE}
terminationGracePeriodSeconds: 29
variant: distroless
priorityClassName: ""
EOF

# Patch the DaemonSet to remove priority class (workaround for chart not respecting the value) if on GKE
kubectl patch daemonset ztunnel -n istio-system -p '{"spec":{"template":{"spec":{"priorityClassName":null}}}}'
```

```
kubectl get pods -n istio-system 
```

## Install Agentgateway

```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml 
```

```
helm upgrade -i agentgateway-crds oci://us-docker.pkg.dev/solo-public/enterprise-agentgateway/charts/enterprise-agentgateway-crds \
  --create-namespace \
  --namespace agentgateway-system \
  --version 2.1.0-rc.2
```

```
helm upgrade -i agentgateway oci://us-docker.pkg.dev/solo-public/enterprise-agentgateway/charts/enterprise-agentgateway \
  -n agentgateway-system \
  --version 2.1.0-rc.2 \
  --set agentgateway.enabled=true \
  --set licensing.licenseKey=${AGENTGATEWAY_LICENSE_KEY}
```

```
kubectl get pods -n agentgateway-system
```

## Install Kagent

```
export KAGENT_MGMT_ENT_VERSION=0.2.0
export KAGENT_ENT_VERSION=0.2.1-nightly-2026-01-11-ae65f848
```

```
helm upgrade -i kagent-mgmt \
oci://us-docker.pkg.dev/solo-public/solo-enterprise-helm/charts/management \
-n kagent --create-namespace \
--version $KAGENT_MGMT_ENT_VERSION \
-f - <<EOF
imagePullSecrets: []
global:
  imagePullPolicy: IfNotPresent
oidc:
  issuer: ${OIDC_ISSUER}
  additionalScopes: #For Azure
    - api://d6957938-c281-4312-97d2-eefbfc44f468/kagent-backend
rbac:
  roleMapping:
    roleMapper: "has(claims.Groups) ? claims.Groups.transformList(i, v, v in rolesMap, rolesMap[v]) : (has(claims.groups) ? claims.groups.transformList(i, v, v in rolesMap, rolesMap[v]) : [])"
    roleMappings:
      966e120a-237f-44fd-9b86-049da1106a93: "global.Admin"
service:
  type: ClusterIP
ui:
  backend:
    oidc:
      clientId: ${OIDC_BACKEND}
      secret: ${BACKEND_CLIENT_SECRET}
  frontend:
    oidc:
      clientId: ${OIDC_FRONTEND}
clickhouse:
  enabled: true
tracing:
  verbose: true
EOF
```

Create TLS certificate and Gateway with Istio:
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/tls.key -out /tmp/tls.crt \
  -subj "/CN=kagent/O=kagent"

kubectl create secret tls kagent-gateway-tls -n kagent \
  --cert=/tmp/tls.crt --key=/tmp/tls.key
```

```
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: kagent-gateway
  namespace: kagent
spec:
  gatewayClassName: istio
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: kagent-gateway-tls
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: kagent-route
  namespace: kagent
spec:
  parentRefs:
  - name: kagent-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: solo-enterprise-ui
      port: 80
EOF
```

Get the LoadBalancer IP:
```
kubectl get svc kagent-gateway-istio -n kagent -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```
Access via `https://<IP>`

```
# Generate the RSA key
openssl genrsa -out /tmp/key.pem 2048
kubectl create secret generic jwt -n kagent --from-file=jwt=/tmp/key.pem
```

```
helm upgrade -i kagent-crds \
oci://us-docker.pkg.dev/developers-369321/kagent-enterprise-public-nonprod/charts/kagent-enterprise-crds \
--version $KAGENT_ENT_VERSION \
-n kagent
```

```
helm upgrade -i kagent \
oci://us-docker.pkg.dev/developers-369321/kagent-enterprise-public-nonprod/charts/kagent-enterprise \
--version $KAGENT_ENT_VERSION \
-n kagent \
-f - <<EOF
oidc:
  clientId: ${OIDC_BACKEND}
  issuer: ${OIDC_ISSUER}
  secretRef: kagent-backend-secret
  secret: ${BACKEND_CLIENT_SECRET}
rbac:
  roleMapping:
    roleMapper: "has(claims.Groups) ? claims.Groups.transformList(i, v, v in rolesMap, rolesMap[v]) : (has(claims.groups) ? claims.groups.transformList(i, v, v in rolesMap, rolesMap[v]) : [])"
    roleMappings:
      966e120a-237f-44fd-9b86-049da1106a93: "global.Admin"
providers:
  default: anthropic
  anthropic:
    apiKey: ${ANTHROPIC_API_KEY}
otel:
  tracing:
    enabled: true
    exporter:
      otlp:
        endpoint: kagent-enterprise-ui.kagent.svc.cluster.local:4317
        insecure: true
controller:
  image:
    repository: kagent-enterprise-public-nonprod
kmcp:
  enabled: true
  image:
    repository: kagent-enterprise-public-nonprod
EOF
```

```
kubectl label namespaces kagent istio.io/dataplane-mode=ambient
```

```
kubectl get pods -n kagent
```

### For Entra

The RBAC configs in kagent-mgmt and kagent need to be updated with your Entra Group Object ID

```
https://learn.microsoft.com/en-us/azure/aks/enable-authentication-microsoft-entra-id#non-interactive-sign-in-with-kubelogin
```

```
brew tap azure/kubelogin && brew install azure/kubelogin/kubelogin 2>&1 || echo "---" && brew search azure/kubelogin
```