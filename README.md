# cloud-apps

Sample Helm charts for Harness deploy testing.

## Charts

| Chart | Path | Description |
|-------|------|-------------|
| `hello-web` | `apps/hello-web` | nginx frontend serving a configurable HTML page |
| `hello-api` | `apps/hello-api` | httpbin-based REST API with health probes |

## Usage

```bash
helm install my-release ./apps/hello-web
helm install my-release ./apps/hello-api
```
