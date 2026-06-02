# Unofficial Hermes Agent Helm Chart

> [!IMPORTANT]
> This is an **unofficial community Helm chart** for [Nous Research's Hermes Agent](https://github.com/nousresearch/hermes-agent). It is maintained independently from the upstream Hermes project.

This chart packages Hermes Agent for Kubernetes with cloud-native defaults, explicit state-safety guardrails, and flexible composition points for platform teams.

## Deployment modes

- **Direct deployment mode** is the current default: Helm renders the Hermes `Deployment`, storage, secrets, service, ingress, Istio, and supporting resources directly.
- **Operator-ready mode** is the API-first path for this pass: the chart can define Hermes tenant/operator CRDs and optional tenant custom resources for clusters that run a compatible controller.

> [!NOTE]
> This chart remains unofficial in both modes. Operator-ready support should be treated as an additive integration surface, not as proof that an operator/controller is bundled here. If your cluster does not already run a compatible controller, stay on direct deployment mode.

## What this chart is optimized for

- **State-safe Hermes deployments**: `replicaCount: 1` plus `strategy.type: Recreate` are enforced when persistence is enabled so `HERMES_HOME` is not shared unsafely.
- **Gateway-first runtime**: the default command is `hermes gateway run`, with optional API server, webhook, and Telegram webhook listeners.
- **Cloud-native integration**: optional Service, Ingress, Istio `VirtualService`, RBAC, NetworkPolicy, PDB, and arbitrary `extraObjects`.
- **Composable secrets and bootstrap**: use chart-managed Secret/ConfigMap resources, or reuse externally managed ones with `secrets.existingSecret` and `bootstrap.existingConfigMap`.
- **Tenant-scoped operation**: the chart is best run as one Helm release per tenant in direct mode, or one tenant custom resource per tenant boundary in operator-ready mode.
- **Explicit unofficial positioning**: README guidance keeps the community-maintained status obvious so platform teams can evaluate risk deliberately.

## Quick start

```bash
helm install hermes . \
  --namespace hermes \
  --create-namespace
```

Minimal values:

```yaml
secrets:
  OPENROUTER_API_KEY: sk-or-...

config:
  values:
    model:
      default: anthropic/claude-opus-4.6
```

## Live kubectl snapshot

This screenshot was captured from a local `kind` cluster (`kind-hermes-test`) after installing a demo release named `hermes-demo` into the `hermes-demo` namespace.

> [!TIP]
> For this live local capture I used `image.tag=latest`. For production deployments, prefer pinning a published Hermes Agent image tag such as the chart default.

![kubectl snapshot from a local Hermes Agent deployment on kind](docs/images/kubectl-kind-hermes-demo.png)

## Core chart behavior

### Hermes state safety

Hermes stores mutable data under `HERMES_HOME`, so this chart treats persistent storage as a **single-writer** workload:

- `persistence.enabled=true` defaults to a PVC mounted at `/opt/data`
- `replicaCount` must remain `1` when persistence is enabled
- `strategy.type` must remain `Recreate` when persistence is enabled
- `persistence.existingClaim` lets you bind to pre-provisioned storage without changing the single-replica rule

If you need multiple Hermes instances, create **multiple releases** instead of scaling a single release horizontally.

### Configuration model

The chart exposes four main configuration layers:

- `config.values`: structured YAML merged into a rendered `config.yaml`
- `config.raw`: raw templated YAML that takes precedence over `config.values`
- `env`, `extraEnv`, and `extraEnvFrom`: environment-level tuning and secrets/config injection
- `command` / `args`: runtime entrypoint overrides when you need something other than the default gateway mode

### Runtime and bootstrap

- `bootstrap.enabled=true` seeds `config.yaml` and optional `SOUL.md` into `HERMES_HOME`
- `bootstrap.overwrite=true` makes Helm the source of truth for bootstrap files
- `bootstrap.existingConfigMap` reuses externally managed bootstrap content instead of rendering a chart-owned ConfigMap
- `npmPackages` installs extra Node packages into the persistent volume and exposes them through `PATH` / `NODE_PATH`

## Operator-ready mode

Operator-ready mode is for platform teams that want Helm to establish the **custom resource API surface** while a separate controller reconciles Hermes tenants. Keep these boundaries in mind:

- Helm/this chart can define the CRDs and optionally render tenant CR objects from values.
- A compatible controller is still required to turn those CRs into running Hermes workloads.
- Direct deployment mode remains the safer default when Helm should continue owning the workload lifecycle end to end.

This repository still does **not** bundle a controller binary. Operator-ready support here means CRDs + CR templates + documentation for use with a separate controller.

## Custom resources and operator-ready flow

The chart now ships a `HermesTenant` CRD (`hermestenants.hermes.ai`) and an optional operator-mode renderer.

Use operator-ready mode when:
- your platform runs a separate controller that reconciles `HermesTenant` resources
- you want a CRD/API surface for tenant lifecycle instead of direct Helm-managed workloads

Key behavior:
- direct mode remains the default
- `operator.enabled=true` renders `HermesTenant` resources from `operator.tenants`
- direct workload templates are suppressed in operator mode so the chart can serve as a CRD/CR producer
- each tenant CR still represents **one Hermes instance per tenant boundary**

Example operator-mode values:

```yaml
operator:
  enabled: true
  controllerClass: hermes.ai/default
  tenants:
    - name: hermes-tenant-a
      namespace: tenant-a
      tenantId: tenant-a
      chartValues:
        tenant:
          id: tenant-a
        apiServer:
          enabled: true
          port: 8642
```

The CRD is intentionally generic: it records tenant metadata, desired release identity, controller class, and arbitrary chart values for a controller to reconcile.

## Multi-tenant isolation pattern

This chart does not model multi-tenancy inside a single Hermes pod. Instead, the safe pattern is **one release per tenant** in direct mode, or one tenant custom resource per tenant boundary in operator-ready mode.

Recommended per-tenant isolation boundaries:

1. **Namespace per tenant** (or equivalent namespace boundary)
2. **Dedicated PVC** per tenant release
3. **Dedicated Secret** or `secrets.existingSecret` per tenant
4. **Dedicated ServiceAccount/RBAC** rules per tenant when Kubernetes API access is needed
5. **Dedicated ingress host / Istio host** per tenant
6. **Tenant-specific NetworkPolicy** to limit ingress and egress

Example tenant-scoped values:

```yaml
fullnameOverride: hermes-tenant-a

serviceAccount:
  create: true
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/hermes-tenant-a

rbac:
  create: true
  rules:
    - apiGroups: [""]
      resources: ["configmaps", "secrets"]
      verbs: ["get", "list", "watch"]

persistence:
  enabled: true
  existingClaim: hermes-tenant-a-data

secrets:
  existingSecret: hermes-tenant-a-secrets

networkPolicy:
  enabled: true
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8642
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

## Secrets and External Secrets Operator patterns

The chart supports two secret-management modes:

1. **Chart-managed Secret** via `secrets.*`
2. **Externally managed Secret** via `secrets.existingSecret`

For External Secrets Operator, the chart now supports two patterns:

- render a first-class `ExternalSecret` with `externalSecret.enabled=true`
- or continue pointing `secrets.existingSecret` at a Secret created by an external operator or another chart

Example using a pre-created Secret:

```yaml
secrets:
  existingSecret: hermes-tenant-a-secrets
```

Example rendering the chart's first-class `ExternalSecret`:

```yaml
externalSecret:
  enabled: true
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: platform-secrets
  target:
    name: hermes-tenant-a-secrets
  data:
    - secretKey: OPENROUTER_API_KEY
      remoteRef:
        key: tenants/tenant-a/hermes
        property: OPENROUTER_API_KEY
```

If your platform already manages the `ExternalSecret` object elsewhere, keep using `secrets.existingSecret` or `extraObjects` and let the chart consume the resulting Kubernetes Secret.

## Service exposure, ingress, and Istio

### OpenAI-compatible API server

Enable Hermes's OpenAI-compatible API server for Open WebUI, LobeChat, LibreChat, or other compatible clients:

```yaml
apiServer:
  enabled: true
  host: 0.0.0.0
  port: 8642

secrets:
  existingSecret: hermes-api-secrets

service:
  enabled: true
```

If `service.ports` is empty, the chart automatically derives Service ports from enabled listeners (`apiServer`, `webhook`, and `telegramWebhook`). Keep explicit `service.ports` only when you need custom front-door mappings.

### Ingress controller pattern

Use `ingress.enabled=true` when you want Kubernetes Ingress resources:

```yaml
service:
  enabled: true

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
  hosts:
    - host: hermes.tenant-a.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - hosts:
        - hermes.tenant-a.example.com
      secretName: hermes-tenant-a-tls
```

Key notes:

- `ingress.enabled=true` requires `service.enabled=true`
- Ingress can target a derived default service port or `ingress.servicePortNumber`
- `service.annotations` can be used for controller- or cloud-specific integration on `LoadBalancer` services

### Istio VirtualService pattern

Use `virtualService.enabled=true` when the cluster is fronted by Istio:

```yaml
service:
  enabled: true

virtualService:
  enabled: true
  gateways:
    - istio-system/public-gateway
  hosts:
    - hermes.tenant-a.example.com
  timeout: 3600s
  servicePortNumber: 8642
```

Validation requires at least one gateway and one host, which keeps broken Istio configs from rendering silently.

## Compose with other platform resources

This chart is intentionally composable instead of prescriptive. Useful integration points include:

- `secrets.existingSecret` for External Secrets, Vault sync, Sealed Secrets, or another chart
- `bootstrap.existingConfigMap` for shared or operator-managed bootstrap content
- `extraEnvFrom` for ConfigMaps/Secrets from another release
- `extraVolumes` and `extraVolumeMounts` for projected credentials or shared storage
- `extraInitContainers` and `extraContainers` for service mesh helpers, bootstrap jobs, or sidecars
- `extraObjects` for `ExternalSecret`, `ServiceMonitor`, policy resources, or tenant-specific control-plane objects
- `serviceAccount.annotations` for IRSA, GKE Workload Identity, or similar cloud IAM bindings

Example external bootstrap reuse:

```yaml
bootstrap:
  enabled: true
  overwrite: false
  existingConfigMap: hermes-shared-bootstrap

secrets:
  existingSecret: hermes-shared-secrets
```

When `bootstrap.existingConfigMap` is set, the referenced ConfigMap must contain `config.yaml` and may optionally include `SOUL.md`.

## Security and operational notes

- Enable `networkPolicy.enabled` to add tenant-scoped ingress and egress controls
- Enable `rbac.create` only when Hermes needs Kubernetes API access
- `serviceAccount.automountServiceAccountToken` defaults to `false`
- Pod and container security contexts default to non-root execution, dropped Linux capabilities, and `RuntimeDefault` seccomp
- Enable `service.enabled` only when you actually need network exposure
- `values.schema.json` validates persistence safety, ingress/service prerequisites, Telegram webhook requirements, and Istio host/gateway inputs before templates render

## Verification

Behavior is locked by Helm lint/template checks plus the regression script in `ci/verify.sh`:

```bash
helm lint .
helm lint . -f ci/test-values.yaml
helm lint . -f ci/existing-claim-values.yaml
helm lint . -f ci/external-bootstrap-values.yaml
helm lint . -f ci/default-service-ports-values.yaml
helm lint . -f ci/external-secret-values.yaml
helm lint . -f ci/tenant-isolation-values.yaml
helm lint . -f ci/operator-values.yaml
helm template hermes .
helm template hermes . -f ci/test-values.yaml
helm template hermes . -f ci/existing-claim-values.yaml
helm template hermes . -f ci/external-bootstrap-values.yaml
helm template hermes . -f ci/default-service-ports-values.yaml
helm template hermes . -f ci/external-secret-values.yaml
helm template hermes . -f ci/tenant-isolation-values.yaml
helm template hermes . --include-crds -f ci/operator-values.yaml
bash ci/verify.sh
```
