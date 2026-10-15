# ktest-gitops

Config base and example Argo CD manifests for the ktest project: a React
frontend, a FastAPI backend (balancing cost charts on Trino) and the
Dagster code locations that import and model the data, all from
[ktest-app](https://github.com/chrismich-k/ktest-app).

```
apps/
  ktest-frontend/
    base/              Deployment, Service, Ingress (/)
    overlays/demo/     example environment
  ktest-api/
    base/              Deployment, Service, Ingress (/api)
    overlays/demo/     example environment
  dagster-code-location/
    base/              Deployment, Service of one code location (gRPC)
    overlays/demo-*/   smard, tradinghub_eu, dataware
  dagster-workspace/
    overlays/demo/     workspace.yaml: one entry per code location
argocd/
  *.yaml.example       Argo CD Applications for the demo overlays
```

The bases contain nothing environment-specific: no namespace, placeholder
image names (`ktest-frontend`, `ktest-api`) and the placeholder host
`ktest.invalid`. An overlay sets all of these.

## Set it up for your cluster

Requirements:

- Kubernetes with an ingress controller, Argo CD and optionally
  cert-manager for TLS, e.g. a cluster set up with
  [hetzner-k3s-hybrid-cluster](https://github.com/chrismich-k/hetzner-k3s-hybrid-cluster)
- for the API: Trino with the views it reads (`api_*`, e.g. in schema
  `dataware_api`). They are created by the dbt assets of the `dataware`
  Dagster code location, from the data the `tradinghub_eu` and `smard`
  code locations import into Iceberg tables.
- for the code locations: Dagster installed with the official Helm chart
  (K8sRunLauncher), reading its workspace from the ConfigMap
  `dagster-workspace` (see `apps/dagster-workspace/overlays/demo`), an
  Iceberg REST catalog with S3 storage and, for dbt, Trino.

1. Fork or copy this repository.
2. Adjust `apps/*/overlays/demo/kustomization.yaml` (values marked
   `CHANGE`): namespace, images, host, ingress class, cert-manager issuer
   and the API's Trino connection. Frontend and API use the same namespace
   and host. For more environments copy `demo` to e.g. `staging`.
3. Create the Trino credentials of the API in that namespace; they do not
   belong into git as plain text:

   ```sh
   kubectl create namespace ktest-demo
   kubectl -n ktest-demo create secret generic api-trino \
     --from-literal=API_TRINO_USER=... \
     --from-literal=API_TRINO_PASSWORD=...
   ```

   To manage them in git instead, encrypt them, e.g. with
   [SOPS](https://github.com/getsops/sops) and
   [KSOPS](https://github.com/viaduct-ai/kustomize-sops) in a separate
   private repository with its own Argo CD Application.
4. Check the rendered manifests: `kustomize build apps/ktest-api/overlays/demo`
5. Copy `argocd/*.yaml.example` to `.yaml`, set your repository URL and
   apply them: `kubectl apply -f argocd/ktest-demo-frontend.yaml -f argocd/ktest-demo-api.yaml`

## Dagster code locations

The base `apps/dagster-code-location/base` runs one code location
(`dagster api grpc`); an overlay per code location sets its name
(`namePrefix: <name>-` plus the label `app.kubernetes.io/instance`), the
image and a ConfigMap with `DAGSTER_MODULE_NAME` and its settings. The run
Jobs the K8sRunLauncher starts get the same image, ConfigMap and Secret
(container context). Each code location needs its credentials as Secret
`<name>-code-location-env` in the Dagster namespace, the keys are listed
at the top of its overlay:

```sh
kubectl -n dagster create secret generic smard-code-location-env \
  --from-literal=SMARD_S3_ACCESS_KEY_ID=... \
  --from-literal=SMARD_S3_SECRET_ACCESS_KEY=...
```

The workspace (`apps/dagster-workspace`) is declared as a whole, with an
entry per code location; it gets its own Application.

## Use the bases from a separate repository

Instead of copying the bases, a (private) repository with only your
overlays can reference them as a remote base, pinned to a tag:

```yaml
resources:
  - https://github.com/chrismich-k/ktest-gitops//apps/ktest-api/base?ref=v0.1.0
```
