# app-helm

Helm chart for deploying **Linker** to Kubernetes. The chart is published as a Helm repository via GitHub Pages and consumed by Argo CD.

Chart repository URL: [https://yahav2305-sailpoint.github.io/app-helm](https://yahav2305-sailpoint.github.io/app-helm)

## Prerequisites

- [kind](https://kind.sigs.k8s.io/) (local cluster)
- [Helm](https://helm.sh/) v3
- [kubectl](https://kubernetes.io/docs/tasks/tools/)

## Testing the chart locally

```sh
# Create a local Kind cluster
kind create cluster --config chart-testing/kind-config.yaml

# Install the chart (default values)
helm install app ./charts/app

# (Optional) install with a custom values file
helm install app ./charts/app --values my-values.yaml

# Clean up
kind delete cluster --name app-testing-cluster
```

## Deploying with Argo CD (local Kind cluster)

This is the recommended end-to-end setup that mirrors production.

1. Create the cluster

    ```sh
    kind create cluster --config chart-testing/kind-config.yaml
    ```

1. Install Argo CD

    ```sh
    helm install argocd oci://ghcr.io/argoproj/argo-helm/argo-cd \
    --namespace argocd --create-namespace --wait
    ```

1. Apply the Argo CD Application manifest

    ```sh
    kubectl apply -f chart-testing/app.yaml
    ```

    This creates an Argo CD `Application` that sources the Helm chart from the published chart repository and the production values from [yahav2305-sailpoint/gitops](https://github.com/yahav2305-sailpoint/gitops). Argo CD will automatically sync and keep the cluster in the desired state.

1. Access the Argo CD UI

    ```sh
    # Get the initial admin password
    kubectl -n argocd get secret argocd-initial-admin-secret \
    -o jsonpath="{.data.password}" | base64 -d

    # Port-forward
    kubectl port-forward -n argocd svc/argocd-server 8080:443
    ```

    Open [http://localhost:8080](http://localhost:8080) — username `admin`, password from the step above (omit the trailing `%`).

1. Access the app

    ```sh
    kubectl port-forward services/app-helm 8081:80
    ```

    Then try:

    ```sh
    curl -s -X POST http://localhost:8081/shorten \
    -H 'Content-Type: application/json' \
    -d '{"url": "https://example.com"}' | jq

    curl http://localhost:8081/health
    curl http://localhost:8081/stats
    ```

1. Clean up

    ```sh
    kind delete cluster --name app-testing-cluster
    ```

## Key chart values

| Value | Default | Description |
| --- | --- | --- |
| `replicaCount` | `3` | Number of pod replicas |
| `image.repository` | `ghcr.io/yahav2305-sailpoint/app` | Container image |
| `image.tag` | `""` (uses `appVersion`) | Image tag override |
| `env.baseUrl` | `http://localhost:8080` | Value of the `BASE_URL` env var inside the pod |
| `ingress.enabled` | `false` | Enable standard Kubernetes Ingress |
| `httpRoute.enabled` | `false` | Enable Gateway API HTTPRoute |
| `autoscaling.enabled` | `false` | Enable HorizontalPodAutoscaler |
| `resources` | `{}` | CPU/memory requests and limits |

See [charts/app/values.yaml](charts/app/values.yaml) for the full reference.

## CI pipeline

| Trigger | Workflow | What happens |
| --- | --- | --- |
| PR to `main` | `pull-request.yaml` | Helm lint → installs chart in an ephemeral Kind cluster |
| Git tag pushed | `prod.yaml` | Packages and publishes the chart to GitHub Pages |

Chart versions are created automatically when [app](https://github.com/yahav2305-sailpoint/app) tags a new release. The `appVersion` in `Chart.yaml` is bumped by the app repo's CI.

## Creating a manual chart release

Normally chart releases are automated by the app repo's CI. To release manually:

```sh
git tag v0.2.0
git push origin v0.2.0
```
