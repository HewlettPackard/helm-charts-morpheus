# Morpheus Worker
The Morpheus Worker can serve as a VDI Gateway <and/or> a Distributed Worker.  Input the required tokens, and update the pointers within the Morpheus UI to be utilized as desired.

## Deploying Morpheus Worker Nodes ##

**Configure Helm Repo**
```console
helm repo add morpheusdata https://gomorpheus.github.io/helm-charts-morpheus/
```
\
**Install Morpheus Worker**
Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`. For example,

```console
helm install morpheus-worker --set replicaCount="1" morpheusdata/morpheus-worker
```
Or supply a Values YAML file and pass as an agrgument as follows:
```console
helm install -f Values.yaml morpheus-worker morpheusdata/morpheus-worker
```

---
## Upgrading Morpheus Worker Nodes ##

There are no persistent items associated with Worker Nodes at this time.  Upgrading is simpling refreshing the repo and performing the following:

```console
helm repo update
helm upgrade -f Values.yaml morpheus-worker morpheusdata/morpheus-worker
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

## Uninstalling Morpheus Worker Chart

To uninstall/delete deployment:
```console
helm uninstall morpheus-worker
```
or
```console
helm delete morpheus-worker --purge
```

The command removes all the Kubernetes components associated with the chart and deletes the release.

## Configuration

The following tables lists the configurable parameters of the Sentry chart and their default Values.

| Parameter                                   | Description                                                                                  | Default                                        |
| ------------------------------------------- | -------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| `image.repository`                            | Image repository                                  | `morpheusdata/morpheus-worker`|
| `image.tag`                                   | Image tag. Possible Values listed [here](https://hub.docker.com/r/morpheusdata/morpheus-worker/tags). | `5.4.3`|
| `image.pullPolicy`                            | Image pull policy | `IfNotPresent`                |                           |
| `env.MORPHEUS_KEY`                            | API Key for Morpheus VDI Gateway                                      |  `<Optional>`             |
| `env.MORPHEUS_WORKER_KEY`                     | API Key for Morpheus Distributed Worker                                |  `<Optional>`             |
| `env.MORPHEUS_URL`                            | Morpheus FQDN with protocol                       |                           |
| `env.MORPHEUS_SELF_SIGNED`                    | Is Morpheus using a Self Signed Certificate       | `false`                   |
| `service.type`                                | Kubernetes service type for the GUI               | `NodePort`               |
| `service.port`                                | Kubernetes port where the GUI is exposed          | `8989`                    |
| `livenessProbe.initialDelaySeconds`           | Initial delay (seconds) for liveness monitoring   | `5`                       |
| `livenessProbe.timeoutSeconds`                | Timeout (seconds) before health check considered unhealthy | `5`              |
| `livenessProbe.periodSeconds`                 | Poll interval (seconds) between health checks     | `10`                      |
| `livenessProbe.failureThreshold`              | Number of failed polls before restarting service  | `3`                       |
| `replicaCount`                                | Number of Replicas if AutoScaling False           | `1`                       |
| `autoscaling.enabled`                         | Enable AutoScaling                                | `false`                   |
| `autoscaling.minReplicas`                     | Minimum number of Replicas                        | `1`                       |
| `autoscaling.maxReplicas`                     | Maximum number of Replicas                        | `100`                     |
| `autoscaling.targetCPUUtilizationPercentage`  | CPU Threshold for AutoScaling                     | `80`                      |
| `autoscaling.targetMemoryUtilizationPercentage`| Memory Threshold for AutoScaling                 | `80`                      |
| `ingress.enabled`                             | Enables Ingress                                   | `false`                   |
| `ingress.annotations`                         | Ingress annotations                               | `{}`                      |
| `ingress.path`                                | Ingress path                                      | `/`                       |
| `ingress.hosts`                               | Ingress accepted hostnames                        | `chart-example.local`     |
| `ingress.tls`                                 | Ingress TLS configuration                         | `[]`                      |
| `resources`                                   | CPU/Memory resource requests/limits               | `{}`                      |
| `nodeSelector`                                | Node labels for pod assignment                    | `{}`                      |
| `tolerations`                                 | Toleration labels for pod assignment              | `[]`                      |
| `affinity`                                    | Affinity settings for pod assignment              | `{}`                      |

---