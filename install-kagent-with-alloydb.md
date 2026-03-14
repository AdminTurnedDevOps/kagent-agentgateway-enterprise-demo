# Single Cluster Install

## Env Vars
```
export ANTHROPIC_API_KEY=
export AGENTGATEWAY_LICENSE_KEY=
```

## Agentgateway Install

```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml 
```

```
helm upgrade -i agentgateway-crds oci://us-docker.pkg.dev/solo-public/enterprise-agentgateway/charts/enterprise-agentgateway-crds \
  --create-namespace \
  --namespace agentgateway-system  \
  --version 2.2.0
```

```
helm upgrade -i agentgateway oci://us-docker.pkg.dev/solo-public/enterprise-agentgateway/charts/enterprise-agentgateway \
  -n agentgateway-system  \
  --version 2.2.0 \
  --set agentgateway.enabled=true \
  --set licensing.licenseKey=${AGENTGATEWAY_LICENSE_KEY}
```

```
kubectl get pods -n agentgateway-system
```

## Kagent Install

```
export KAGENT_MGMT_ENT_VERSION=0.3.9-nightly-2026-03-13-eaa3f022
export KAGENT_ENT_VERSION=0.3.9-nightly-2026-03-13-eaa3f022
```

```
helm upgrade -i kagent-mgmt \
oci://us-docker.pkg.dev/developers-369321/solo-enterprise-public-nonprod/charts/management \
-n kagent --create-namespace \
--version $KAGENT_MGMT_ENT_VERSION \
-f - <<EOF
products:
  kagent:
    enabled: true
  agentgateway:
    enabled: true
    namespace: agentgateway-system
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
    oidc:
      clientId: ${OIDC_BACKEND}
      secret: ${BACKEND_CLIENT_SECRET}
    extraEnvs:
      OIDC_INSECURE_SKIP_VERIFY:
        value: "true"
  frontend:
    oidc:
      clientId: ${OIDC_FRONTEND}
clickhouse:
  enabled: true
traces:
  verbose: true
EOF
```

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
        endpoint: solo-enterprise-telemetry-collector.kagent.svc.cluster.local:4317
        insecure: true
database:
  type: postgres
  postgres:
    url: postgres://postgres:YOUR_PASSWORD@YOUR_ALLOYDB_HOST_OR_IP:5432/YOUR_DATABASE
EOF
```