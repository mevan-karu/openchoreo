# Okta Identity Provider — k3d Tryout

This directory contains values files for running the OpenChoreo k3d single-cluster setup with Okta as the identity provider instead of the default Thunder.

Before using these values, complete the Okta application and authorization server setup described in [docs/configure-okta-idp.md](../../../docs/configure-okta-idp.md). You will need the client IDs, client secrets, and authorization server URLs from those steps to fill in the placeholders below.

## 1. Fill in the Placeholders

Edit both values files and replace every `<placeholder>` with your actual values:

| Placeholder | Where to find it |
|---|---|
| `<your-okta-domain>` | Your Okta domain, e.g. `dev-1234567.okta.com` |
| `<authorization-server-id>` | Authorization server ID from the Okta API settings, e.g. `aus1ab2cd3ef4gh5ij` |
| `<okta-authorization-server-audience>` | Audience set on the Okta authorization server in Step 1 of the guide |
| `<okta-backstage-client-id>` | Client ID of the Backstage web application |
| `<okta-cli-client-id>` | Client ID of the CLI native application |
| `<okta-observer-client-id>` | Client ID of the Observer Resource Reader API Services application |

## 2. Follow the Single-Cluster Setup

Follow [install/k3d/single-cluster/README.md](../single-cluster/README.md) with the following differences:

### Skip Thunder

Do not install Thunder. Jump directly to the CoreDNS Rewrite step.

### Create the Backstage Secret with the Okta Client Secret

When creating `backstage-secrets`, use the Okta Backstage application's client secret:

```bash
kubectl create namespace openchoreo-control-plane --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic backstage-secrets \
  -n openchoreo-control-plane \
  --from-literal=backend-secret="$(head -c 32 /dev/urandom | base64)" \
  --from-literal=client-secret="<okta-backstage-client-secret>" \
  --from-literal=jenkins-api-key="placeholder-not-in-use"
```

### Seed the Observer Client Secret in OpenBao

After running `install/prerequisites/openbao/setup.sh`, store the Okta Observer application's client secret so the uid-resolver can retrieve it:

```bash
kubectl exec -n openbao openbao-0 -- \
  env BAO_ADDR=http://127.0.0.1:8200 BAO_TOKEN=root \
  bao kv put secret/observer-oauth-client-secret value="<okta-observer-client-secret>"
```

### Use These Values Files

When installing the control plane and observability plane, pass the values files from this directory instead of the single-cluster ones:

```bash
# Control plane
helm upgrade --install openchoreo-control-plane install/helm/openchoreo-control-plane \
  --namespace openchoreo-control-plane \
  --create-namespace \
  --values install/k3d/okta/values-cp.yaml

# Observability plane
helm upgrade --install openchoreo-observability-plane install/helm/openchoreo-observability-plane \
  --dependency-update \
  --namespace openchoreo-observability-plane \
  --values install/k3d/okta/values-op.yaml \
  --timeout 10m
```

All other steps (data plane, workflow plane, port mappings, verification) are identical to the single-cluster README.
