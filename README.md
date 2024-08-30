# Flux Experiments

This repository contains the setup and configuration for deploying a simple HTTP server application named Pong to a Kubernetes cluster using Flux CD.

## Prerequisites

Before getting started, make sure you have the following installed locally:

- A Kubernetes cluster (in this experiment we use [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/))
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Flux CLI](https://fluxcd.io/flux/cmd/)
- GitHub Personal Access Token with repo access
- [GitHub CLI](https://cli.github.com/)

## Preparing the Git Repository

To prepare your Git repository, follow these steps:

1. Create a new directory and initialize a Git repository:

    ```bash
    mkdir flux-experiments
    cd flux-experiments
    git init
    ```

2. Create a `manifests` directory to store Kubernetes manifests:

    ```bash
    mkdir manifests
    ```

## Creating Deployment and Service Manifests

Create the following manifest files in the `manifests` directory:

1. **Deployment Manifest**: Create `pong-deployment.yaml`:

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: pong-server-deployment
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: pong-server
      template:
        metadata:
          labels:
            app: pong-server
        spec:
          containers:
          - name: pong-server
            image: ghcr.io/s1ntaxe770r/pong:e0fb83f27536836d1420cffd0724360a7a650c13
            ports:
            - containerPort: 8080
    ```

2. **Service Manifest**: Create `pong-service.yaml`:

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: pong-server-service
    spec:
      selector:
        app: pong-server
      ports:
        - protocol: TCP
          port: 80
          targetPort: 8080
    ```

3. Add and commit these files to Git:

    ```bash
    git add .
    git commit -m "Add deployment manifests"
    ```

4. Create the GitHub repository and push your changes:

    ```bash
    gh repo create flux-experiments --source=. --remote=upstream --public
    git push --set-upstream upstream master
    ```

## Bootstrapping Flux

To bootstrap Flux and configure it to monitor your GitHub repository:

1. Run pre-installation checks:

    ```bash
    flux check --pre
    ```

2. Export your GitHub credentials:

    ```bash
    export GITHUB_TOKEN=<your-token>
    export GITHUB_USER=<your-username>
    ```

3. Run the bootstrap command:

    ```bash
    flux bootstrap github \
      --owner=$GITHUB_USER \
      --repository=flux-experiments \
      --branch=master \
      --path=clusters/my-cluster
    ```

4. Verify Flux installation:

    ```bash
    kubectl get pods -n flux-system
    ```

5. Pull the latest changes:

    ```bash
    git pull
    ```

## Automating Deployments with Flux

To configure Flux to deploy your manifests:

1. Define a `GitRepository` resource:

    ```bash
    flux create source git pong \
      --url="https://github.com/$GITHUB_USER/flux-experiments" \
      --branch=master \
      --interval=30s \
      --export > ./clusters/my-cluster/pong-source.yaml
    ```

2. Define a `Kustomization` resource:

    ```bash
    flux create kustomization pong \
      --target-namespace=default \
      --source=pong \
      --path="./manifests" \
      --prune=true \
      --interval=5m \
      --export > ./clusters/my-cluster/pong-kustomization.yaml
    ```

3. Inspect the generated manifest:

    ```bash
    cat ./clusters/my-cluster/pong-kustomization.yaml
    ```

4. Commit and push the generated manifests:

    ```bash
    git add clusters/
    git commit -m "Add kustomize manifests"
    git push
    ```

5. Watch Flux sync the application:

    ```bash
    flux get kustomizations --watch
    ```

6. Verify deployment:

    ```bash
    kubectl port-forward svc/pong-server-service 8080:80
    curl http://localhost:8080/ping
    ```

## Updating the Pong App with Flux

To update the deployment:

1. Edit the `pong-deployment.yaml` file to increase replicas:

    ```yaml
    replicas: 3
    ```

2. Save, commit, and push the changes:

    ```bash
    git add pong-deployment.yaml
    git commit -m "Scale pong deployment to 3 replicas"
    git push origin main
    ```

3. Verify the update:

    ```bash
    flux logs
    kubectl get pods
    ```

## Troubleshooting

- **Service Not Found**: If you encounter `services "pong-server-service" not found`, verify that the service is correctly defined and deployed.
- **Connection Issues**: Ensure the port-forwarding is correctly set up and check if the application is running properly inside the pods.

Feel free to open issues if you encounter any problems or have questions!

