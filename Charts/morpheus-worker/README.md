# Morpheus Worker

Deploys [HPE Morpheus](https://www.morpheusdata.com) worker nodes that can serve as a **VDI Gateway** and/or a **Distributed Worker**. Supply the required API keys, then update the corresponding pointers in the Morpheus UI.

## Prerequisites

- Kubernetes 1.23+
- Helm 3
- Only when enabling `httpRoute`: the [Gateway API](https://gateway-api.sigs.k8s.io/) standard-channel CRDs and an existing `Gateway`

## Installing

```console
helm repo add morpheusdata https://hewlettpackard.github.io/helm-charts-morpheus/
helm repo update
```

Install with parameters set on the command line:

```console
helm install morpheus-worker morpheusdata/morpheus-worker \
  --set env.MORPHEUS_URL="https://morpheus.example.com" \
  --set env.MORPHEUS_WORKER_KEY="<key>"
```

Or supply a values file:

```console
helm install -f values.yaml morpheus-worker morpheusdata/morpheus-worker
```

## Upgrading

Worker nodes hold no persistent state; upgrading is a repo refresh plus:

```console
helm repo update
helm upgrade -f values.yaml morpheus-worker morpheusdata/morpheus-worker
```

## Upgrading to 2.0.0 (breaking changes)

Chart 2.0.0 reworks the ingress values schema and requires Kubernetes 1.23+. Rendering fails with an explicit error message when 1.x-era keys are supplied. To stay on the old schema, pin chart `1.4.1` (`helm upgrade --version 1.4.1 ...`).

**Ingress values schema.** `ingress.path` and host-only `ingress.hosts[]` entries were removed in favor of the standard per-host paths shape, and the deprecated `kubernetes.io/ingress.class` annotation was replaced by `ingress.className`:

```yaml
# 1.x shape (rejected by 2.0.0)
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
  hosts:
    - host: morpheus-worker.example.com
  path: /

# 2.0.0 shape
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: morpheus-worker.example.com
      paths:
        - path: /
          pathType: Prefix
```

**Controller-neutral defaults.** `ingress.annotations` now defaults to `{}` — the chart no longer ships nginx-specific annotations, and the hardcoded `nginx.org/websocket-services` annotation (which pointed at a nonexistent Service name) is no longer rendered by the template. Copy the commented example matching your controller from `values.yaml`, replacing `<fullname>` with the rendered Service name (default: `<release>-morpheus-worker`):

```yaml
# kubernetes/ingress-nginx
ingress:
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"   # only when worker.protocol=https
    nginx.ingress.kubernetes.io/forwarded-for-header: "X-Forwarded-For"
    nginx.ingress.kubernetes.io/x-forwarded-proto: "https"
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/affinity-mode: "persistent"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"  # long-lived websocket sessions
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"

# NGINX Inc. kubernetes-ingress
ingress:
  annotations:
    nginx.org/websocket-services: "<fullname>"              # MUST be the full Service name
    nginx.com/sticky-cookie-services: "serviceName=<fullname> srv_id expires=1h path=/"
    nginx.org/proxy-read-timeout: "3600"
    nginx.org/proxy-send-timeout: "3600"
    nginx.org/ssl-services: "<fullname>"                    # only when worker.protocol=https
```

**Worker protocol.** A new `worker.protocol` value (default `http`, port 8080) drives the liveness/readiness probe scheme. HTTPS deployments set `worker.protocol=https`, `service.ports.targetPort=8443`, and the backend-protocol (or `nginx.org/ssl-services`) annotation for their controller.

**Probe target.** Liveness/readiness probes now target the container port (`service.ports.targetPort`) instead of the Service port. With default values (both 8080) there is no behavior change; if you had diverged the two values, the probe now targets the correct port.

## Exposing the worker

The chart supports classic Ingress and Gateway API as independent toggles — enable either, or both during a migration.

### Classic Ingress

```yaml
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: morpheus-worker.example.com
      paths:
        - path: /
          pathType: Prefix
```

No controller-specific annotations ship by default; see the commented examples in `values.yaml` (reproduced in the 2.0.0 section above).

### Gateway API (HTTPRoute)

The chart can expose the worker via the [Gateway API](https://gateway-api.sigs.k8s.io/) instead of, or alongside, classic Ingress.

**Prerequisites:** the Gateway API standard-channel CRDs must be installed in the cluster, and a `Gateway` must already exist — this chart does not create one. `httpRoute.parentRefs` must reference it; rendering fails with an explicit error otherwise.

```yaml
httpRoute:
  enabled: true
  parentRefs:
    - name: my-gateway
      namespace: my-gateway-namespace
      sectionName: http
  hostnames:
    - morpheus-worker.example.com
```

Each entry in `httpRoute.rules[]` renders its `matches` and `filters` verbatim; the `backendRefs` are always chart-owned and target the worker Service on `service.ports.port`.

**Behaviors that do not port from the Ingress annotations:**
- *Cookie session affinity* (`nginx.ingress.kubernetes.io/affinity` and friends) has no portable Gateway API equivalent — session persistence (`BackendLBPolicy`) is still experimental. Configure it per-implementation on your Gateway if you need it.
- *Backend TLS* (when `worker.protocol=https`): controller annotations like `backend-protocol: HTTPS` do not apply to HTTPRoute. Gateway API models this as a separate user-supplied [`BackendTLSPolicy`](https://gateway-api.sigs.k8s.io/api-types/backendtlspolicy/) resource, which this chart does not ship; some implementations offer their own alternatives.

## Uninstalling

```console
helm uninstall morpheus-worker
```

This removes all Kubernetes components associated with the chart and deletes the release.

## Configuration

The following table lists the configurable parameters of the morpheus-worker chart and their default values.

| Parameter | Description | Default |
| --------- | ----------- | ------- |
| `replicaCount` | Number of replicas when autoscaling is disabled | `1` |
| `env.MORPHEUS_KEY` | API key for the Morpheus VDI Gateway (optional) | `""` |
| `env.MORPHEUS_WORKER_KEY` | API key for the Morpheus Distributed Worker (optional) | `""` |
| `env.MORPHEUS_URL` | Morpheus FQDN, including protocol | `""` |
| `env.MORPHEUS_SELF_SIGNED` | Whether Morpheus uses a self-signed certificate | `"false"` |
| `image.repository` | Image repository | `morpheusdata/morpheus-worker` |
| `image.tag` | Image tag ([available tags](https://hub.docker.com/r/morpheusdata/morpheus-worker/tags)); defaults to the chart `appVersion` when empty | `""` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `nameOverride` | Override the chart name | `""` |
| `fullnameOverride` | Override the fully qualified release name | `""` |
| `podAnnotations` | Annotations added to worker pods | `{}` |
| `worker.protocol` | Protocol the worker listens on (`http` or `https`); drives the probe scheme | `http` |
| `service.type` | Kubernetes Service type | `NodePort` |
| `service.ports.port` | Service port | `8080` |
| `service.ports.targetPort` | Container port the worker listens on | `8080` |
| `service.ports.nodePort` | Fixed NodePort (only honored when `service.type` is `NodePort`) | unset |
| `livenessProbe.initialDelaySeconds` | Liveness probe initial delay (seconds) | `5` |
| `livenessProbe.timeoutSeconds` | Liveness probe timeout (seconds) | `5` |
| `livenessProbe.periodSeconds` | Liveness probe poll interval (seconds) | `10` |
| `livenessProbe.failureThreshold` | Failed liveness polls before restart | `3` |
| `readinessProbe.initialDelaySeconds` | Readiness probe initial delay (seconds) | `5` |
| `readinessProbe.timeoutSeconds` | Readiness probe timeout (seconds) | `5` |
| `readinessProbe.periodSeconds` | Readiness probe poll interval (seconds) | `10` |
| `readinessProbe.failureThreshold` | Failed readiness polls before unready | `3` |
| `ingress.enabled` | Enable classic Ingress | `false` |
| `ingress.className` | Ingress controller class (`spec.ingressClassName`) | `""` |
| `ingress.annotations` | Ingress annotations (controller-specific examples in `values.yaml`) | `{}` |
| `ingress.hosts` | Hosts, each with `host` and `paths[].{path,pathType}` | `[{host: chart-example.local, paths: [{path: /, pathType: Prefix}]}]` |
| `ingress.tls` | Ingress TLS configuration | `[]` |
| `httpRoute.enabled` | Enable Gateway API HTTPRoute | `false` |
| `httpRoute.annotations` | HTTPRoute annotations | `{}` |
| `httpRoute.parentRefs` | References to an existing Gateway (required when enabled) | `[]` |
| `httpRoute.hostnames` | HTTPRoute hostnames | `[chart-example.local]` |
| `httpRoute.rules` | Rules (`matches`/`filters` pass through; `backendRefs` are chart-owned) | one `PathPrefix /` match |
| `resources` | CPU/memory requests and limits | memory: `512Mi` request, `2Gi` limit |
| `autoscaling.enabled` | Enable the HorizontalPodAutoscaler | `false` |
| `autoscaling.minReplicas` | Minimum replicas | `1` |
| `autoscaling.maxReplicas` | Maximum replicas | `100` |
| `autoscaling.targetCPUUtilizationPercentage` | CPU threshold for autoscaling | `80` |
| `autoscaling.targetMemoryUtilizationPercentage` | Memory threshold for autoscaling | `80` |
| `nodeSelector` | Node labels for pod assignment | `{}` |
| `tolerations` | Tolerations for pod assignment | `[]` |
| `affinity` | Affinity rules for pod assignment | `{}` |
