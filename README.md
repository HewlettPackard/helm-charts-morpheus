# Morpheus Helm Charts

Helm charts for [HPE Morpheus](https://www.morpheusdata.com), published via GitHub Pages.

- **Source repo:** https://github.com/HewlettPackard/helm-charts-morpheus
- **Charts repo:** https://hewlettpackard.github.io/helm-charts-morpheus/
- **Worker image:** https://hub.docker.com/r/morpheusdata/morpheus-worker

## Usage

```console
helm repo add morpheusdata https://hewlettpackard.github.io/helm-charts-morpheus/
helm repo update
helm search repo morpheusdata
```

## Charts

| Chart | Description |
| ----- | ----------- |
| [morpheus-worker](Charts/morpheus-worker/) | Morpheus VDI Gateway and/or Distributed Worker nodes |
| [redoc](Charts/redoc/) | [Redoc](https://github.com/Redocly/redoc) OpenAPI documentation server |
