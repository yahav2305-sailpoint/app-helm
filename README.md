# app-helm

## Testing the helm chart locally

1. Install Kind on host:

    ```sh
    brew install kind
    ```

1. Start up a Kind cluster:

    ```sh
    kind create cluster --config chart-testing/kind-config.yaml
    ```

1. Install the helm chart on the cluster:

    ```sh
    helm install app ./charts/app --values <your-values-file>
    ```

1. Once you are done, delete the Kind cluster:

    ```sh
    kind delete cluster --name app-testing-cluster
    ```

## Deploying the chart in a production Kind cluster

1. Install Kind on host:

    ```sh
    brew install kind
    ```

1. Start up a Kind cluster:

    ```sh
    kind create cluster --config chart-testing/kind-config.yaml
    ```

1. Install ArgoCD on the Kind cluster:

    ```sh
    helm install argocd oci://ghcr.io/argoproj/argo-helm/argo-cd --namespace argocd --create-namespace --wait
    ```

1. Install the app chart through GitOps:

    ```sh
    kubectl apply -f chart-testing/app.yaml
    ```

1. Get the admin password:

    ```sh
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
    ```

1. Connect to the ArgoCD UI:

    ```sh
    kubectl port-forward -n argocd svc/argocd-server -n argocd 8080:443
    ```

    Then go to [http://localhost:8080](http://localhost:8080) and use the username `admin` and the password from the previous step (without the % at the end).

    Now you can make sure that everything is working correctly.

1. Connect to the app:

    ```sh
    kubectl port-forward services/app-helm 8081:80
    ```

    Now you can interact with the app at [http://localhost:8081](http://localhost:8081).

1. Once you are done, delete the Kind cluster:

    ```sh
    kind delete cluster --name app-testing-cluster
    ```

## Creating a new version

In order to create a new version of the helm chart, make the required changes (whether in the main branch or by merging feature branches to main) and then create a new release with a tag that has a higher semver than the previous release.\
New versions will autoamtically be created for new docker images.
