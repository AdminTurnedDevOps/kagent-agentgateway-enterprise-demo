# Single Cluster Install

## Env Vars
```
export GLOO_GATEWAY_LICENSE_KEY=

export AGENTGATEWAY_LICENSE_KEY=

export ANTHROPIC_API_KEY=
```

## Ambient Config
```
export ISTIO_VERSION=1.27.0
export ISTIO_IMAGE=${ISTIO_VERSION}-solo
export REPO_KEY=
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
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml 
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

# Patch the DaemonSet to remove priority class (workaround for chart not respecting the value)
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

# Patch the DaemonSet to remove priority class (workaround for chart not respecting the value)
kubectl patch daemonset ztunnel -n istio-system -p '{"spec":{"template":{"spec":{"priorityClassName":null}}}}'
```

```
kubectl get pods -n istio-system 
```

## Install Gloo Gateway

```
helm upgrade -i gloo-gateway-crds oci://us-docker.pkg.dev/solo-public/gloo-gateway/charts/gloo-gateway-crds \
  --create-namespace \
  --namespace gloo-system \
  --version 2.0.1
```

```
helm upgrade -i gloo-gateway oci://us-docker.pkg.dev/solo-public/gloo-gateway/charts/gloo-gateway \
  -n gloo-system \
  --version 2.0.1 \
  --set agentgateway.enabled=true \
  --set licensing.agentgatewayLicenseKey=${AGENTGATEWAY_LICENSE_KEY} \
  --set licensing.glooGatewayLicenseKey=${GLOO_GATEWAY_LICENSE_KEY}
```

```
kubectl get pods -n gloo-system
```

## Install Kagent

```
export KAGENT_ENT_VERSION=0.1.10-2025-12-09-main-1d5ad1ac
```

```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: kagent-backend-secret
  namespace: kagent
type: Opaque
stringData:
  clientSecret: ${BACKEND_CLIENT_SECRET}
  secret: ${BACKEND_CLIENT_SECRET}
EOF
```

```
helm upgrade -i kagent-mgmt \
oci://us-docker.pkg.dev/developers-369321/kagent-enterprise-public-nonprod/charts/management \
-n kagent --create-namespace \
--version $KAGENT_ENT_VERSION \
-f - <<EOF
imagePullSecrets: []
global:
  imagePullPolicy: IfNotPresent
oidc:
  issuer: ${OIDC_ISSUER}
rbac:
  roleMapping:
    roleMapper: "claims.Groups.transformList(i, v, v in rolesMap, rolesMap[v])"
    roleMappings:
      admins: "global.Admin"
      readers: "global.Reader"
      writers: "global.Writer"
service:
  type: LoadBalancer
  clusterIP: ""
ui:
  backend:
    repository: kagent-enterprise-public-nonprod
    oidc:
      clientId: ${OIDC_BACKEND}
      secret: ${BACKEND_CLIENT_SECRET}
  frontend:
    repository: kagent-enterprise-public-nonprod
    oidc:
      clientId: ${OIDC_FRONTEND}
tunnelserver:
  repository: kagent-enterprise-public-nonprod
clickhouse:
  enabled: true
tracing:
  verbose: true
EOF
```

```
openssl genrsa -out /tmp/key.pem 2048

  kubectl create secret generic jwt \
  -n kagent --context ${context} \
  --from-file=jwt=/tmp/key.pem \
  --dry-run=client -o yaml | kubectl apply --context ${context} -f -
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
    roleMapper: "claims.Groups.transformList(i, v, v in rolesMap, rolesMap[v])"
    roleMappings:
      admins: "global.Admin"
      readers: "global.Reader"
      writers: "global.Writer"
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
kubectl get pods -n kagent
```