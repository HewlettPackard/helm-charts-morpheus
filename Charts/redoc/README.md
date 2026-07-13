# Redoc

Deploys [Redoc](https://github.com/Redocly/redoc), an open-source tool for generating API documentation from OpenAPI (formerly Swagger) definitions, using the [redocly/redoc](https://hub.docker.com/r/redocly/redoc/) container.

This chart wraps the [k8s-at-home common library chart](https://github.com/k8s-at-home/library-charts) (v4.5.2) — any common-chart value (`controller.replicas`, `podAnnotations`, additional services/ingresses, etc.) can be set alongside the values documented below. The upstream k8s-at-home project is archived; the library chart is vendored in `charts/` so this chart remains buildable regardless.

## Prerequisites

- Kubernetes 1.23+
- Helm 3
- Only when enabling `httpRoute`: the [Gateway API](https://gateway-api.sigs.k8s.io/) standard-channel CRDs and an existing `Gateway`

## Installing

```console
helm repo add morpheusdata https://hewlettpackard.github.io/helm-charts-morpheus/
helm repo update
```

Point the chart at your OpenAPI document and install:

```console
helm install redoc morpheusdata/redoc \
  --set env.SPEC_URL="https://example.com/openapi.json"
```

Or supply a values file:

```console
helm install -f values.yaml redoc morpheusdata/redoc
```

## Upgrading

Redoc holds no persistent state; upgrading is a repo refresh plus:

```console
helm repo update
helm upgrade -f values.yaml redoc morpheusdata/redoc
```

## Upgrading to 2.0.0 (breaking changes)

Chart 2.0.0 fixes defects that made 1.x non-functional and requires Kubernetes 1.23+. Rendering fails with an explicit error message when 1.x-era keys are supplied. To stay on the old chart, pin `1.2.0` (`helm upgrade --version 1.2.0 ...`).

**Configuration now reaches the container.** The 1.x `redoc.*` values block (SPEC_URL, PAGE_TITLE, …) was never delivered to the container — every documented knob was a silent no-op. It has been removed; set environment variables under `env:` instead:

```yaml
# 1.x shape (rejected by 2.0.0 — and it never worked)
redoc:
  SPEC_URL: "https://example.com/openapi.json"
  PAGE_TITLE: "My API"

# 2.0.0 shape
env:
  SPEC_URL: "https://example.com/openapi.json"
  PAGE_TITLE: "My API"
```

**Ports fixed.** 1.x pointed the container port, Service, and readiness/startup probes at 8008 while redoc's nginx listens on 80 — default installs never became ready. 2.0.0 keeps the external Service port 8008 and adds `targetPort: 80`; the custom `probes.liveness` override is gone (the library defaults now probe the correct port).

**Ingress values schema.** `ingress.main.path` and host-only `ingress.main.hosts[]` entries were removed (in 1.x, enabling ingress crashed rendering); use the per-host paths shape, and `ingress.main.ingressClassName` instead of the `kubernetes.io/ingress.class` annotation:

```yaml
ingress:
  main:
    enabled: true
    ingressClassName: nginx
    hosts:
      - host: docs.example.com
        paths:
          - path: /
            pathType: Prefix
```

**Controller-neutral defaults.** `ingress.main.annotations` now defaults to `{}` — the nginx CORS set (and the incorrect `backend-protocol: HTTPS`; redoc serves plain HTTP) no longer ships. See the commented example in `values.yaml`.

**Metrics stack removed.** The 1.x exportarr sidecar, ServiceMonitor, and PrometheusRule could never function — exportarr does not support redoc and the container exposes no metrics endpoint. `metrics.*` and `persistence.config` values are rejected with an explanatory error.

**Other removals.** `replicaCount` was never consumed — use `controller.replicas`.

## Exposing redoc

Classic Ingress (`ingress.main`, above) and Gateway API are independent toggles — enable either, or both during a migration.

### Gateway API (HTTPRoute)

**Prerequisites:** the Gateway API standard-channel CRDs must be installed in the cluster, and a `Gateway` must already exist — this chart does not create one. `httpRoute.parentRefs` must reference it; rendering fails with an explicit error otherwise.

```yaml
httpRoute:
  enabled: true
  parentRefs:
    - name: my-gateway
      namespace: my-gateway-namespace
      sectionName: http
  hostnames:
    - docs.example.com
```

Each entry in `httpRoute.rules[]` renders its `matches` and `filters` verbatim; the `backendRefs` are always chart-owned and target the redoc Service on `service.main.ports.http.port`.

## Uninstalling

```console
helm uninstall redoc
```

This removes all Kubernetes components associated with the chart and deletes the release.

## Configuration

The following table lists the chart-specific parameters and their default values. Any [common library chart](https://github.com/k8s-at-home/library-charts) value is also accepted.

| Parameter | Description | Default |
| --------- | ----------- | ------- |
| `image.repository` | Image repository | `redocly/redoc` |
| `image.tag` | Image tag ([available tags](https://hub.docker.com/r/redocly/redoc/tags)) | `v2.5.3` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `env.SPEC_URL` | URL of the OpenAPI document to render | `https://petstore.swagger.io/v2/swagger.json` |
| `env` | Additional redoc environment variables (`PAGE_TITLE`, `PAGE_FAVICON`, `REDOC_OPTIONS`, …) | see values.yaml |
| `service.main.ports.http.port` | Service port | `8008` |
| `service.main.ports.http.targetPort` | Container port (redoc's nginx listens on 80) | `80` |
| `ingress.main.enabled` | Enable classic Ingress | `false` |
| `ingress.main.ingressClassName` | Ingress controller class (`spec.ingressClassName`) | unset |
| `ingress.main.annotations` | Ingress annotations (CORS example in `values.yaml`) | `{}` |
| `ingress.main.hosts` | Hosts, each with `host` and `paths[].{path,pathType}` | `[{host: chart-example.local, paths: [{path: /, pathType: Prefix}]}]` |
| `ingress.main.tls` | Ingress TLS configuration | `[]` |
| `httpRoute.enabled` | Enable Gateway API HTTPRoute | `false` |
| `httpRoute.annotations` | HTTPRoute annotations | `{}` |
| `httpRoute.parentRefs` | References to an existing Gateway (required when enabled) | `[]` |
| `httpRoute.hostnames` | HTTPRoute hostnames | `[chart-example.local]` |
| `httpRoute.rules` | Rules (`matches`/`filters` pass through; `backendRefs` are chart-owned) | one `PathPrefix /` match |
