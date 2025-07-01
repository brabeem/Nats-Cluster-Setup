# Kubernetes Deployment Guide

This guide provides step-by-step instructions to deploy the complete application stack, including a high-availability Redis cluster, a PostgreSQL database, local and cloud NATS servers, and the custom `nats-auth-callout-server` service.

## Prerequisites

1.  **Docker Desktop**: Installed locally to build and push container images.
2.  **Docker Hub Account**: A free account on [Docker Hub](https://hub.docker.com/) or another container registry.
3.  **`kubectl`**: Installed and configured to connect to your remote Kubernetes cluster. Verify with `kubectl get nodes`.
4.  **`helm`**: Helm v3 installed locally. Verify with `helm version`.

---

## Step 1: Build and Push the `nats-auth-callout-server` Docker Image

The remote Kubernetes cluster cannot use your local Docker images. You must build the `nats-auth-callout-server` image and push it to a container registry that the cluster can access.

1.  **Log in to Docker Hub**:
    Open a terminal and run the following command. You will be prompted for your Docker Hub username and password (or an access token).
    ```bash
    docker login
    ```

2.  **Build the Image**:
    Navigate to the `nats-auth-callout-server` directory. Build the Docker image, replacing `your-dockerhub-username` with your actual username.
    ```bash
    # Make sure you are in the auth-callout directory
    cd nats-auth-callout-server 
    docker build -t your-dockerhub-username/nats-auth-callout-server:latest .
    cd .. 
    ```

3.  **Push the Image to Docker Hub**:
    ```bash
    docker push your-dockerhub-username/nats-auth-callout-server:latest
    ```

---

## Step 2: Update Deployment for the Remote Image

Now, modify the `nats-auth-callout-server` deployment manifest to use the image you just pushed.

1.  **Edit `auth-callout-deployment.yaml`**:
    Open `/auth-callout/auth-callout-deployment.yaml`. Find the `image` key and change its value from the local name to your full Docker Hub image name.

    ````yaml
    // filepath: nats-cluster/auth-callout/auth-callout-deployment.yaml
    // ...existing code...
    spec:
      containers:
        - name: auth-callout-server
          # Change this line to point to your Docker Hub image
          image: your-dockerhub-username/nats-auth-callout-server:latest
          imagePullPolicy: Always # Ensures the latest image is always pulled
    // ...existing code...
    ````

2.  **(Optional) Create an Image Pull Secret**:
    If your Docker Hub repository is private, you must create a secret in your Kubernetes cluster so it can authenticate and pull the image. If your repository is public, you can skip this step.
    ```bash
    kubectl create secret docker-registry regcred \
      --docker-server=https://index.docker.io/v1/ \
      --docker-username=your-dockerhub-username \
      --docker-password=your-password-or-token \
      --docker-email=your-email
    ```
    Then, add the secret to your deployment manifest:
    ````yaml
    // filepath: nats-cluster/auth-callout/auth-callout-deployment.yaml
    // ...existing code...
    spec:
      # Add this section to use the secret
      imagePullSecrets:
        - name: regcred
      containers:
    // ...existing code...
    ````

---

## Step 3: Deploy the Infrastructure

Deploy all components in the correct order.

1.  **Deploy PostgreSQL**:
    This deploys the database service, deployment, and persistent volume claim.
    ```bash
    kubectl apply -f ./postgres/
    ```

2.  **Deploy Redis with Sentinel**:
    First, add the Bitnami Helm repository. Then, install Redis using the provided values file for a high-availability configuration.
    ```bash
    helm repo add bitnami https://charts.bitnami.com/bitnami
    helm repo update
    helm install redis bitnami/redis -f ./redis/my-redis-values.yaml
    ```

3.  **Deploy Local NATS**:
    This is the simple NATS server for internal administrative tasks.
    ```bash
    kubectl apply -f ./nats-local/
    ```

4.  **Deploy the Main NATS Cluster**:
    This deploys the main, client-facing NATS cluster using the official NATS Helm chart.
    ```bash
    helm repo add nats https://nats-io.github.io/k8s/helm/charts/
    helm repo update
    helm install nats nats/nats -f ./nats-cloud/nats-helm-values.yaml
    ```

---

## Step 4: Deploy the `auth-callout` Application

With all dependencies running, deploy the `auth-callout` server.

```bash
kubectl apply -f ./auth-callout/
```

Wait for the pod to be in the `Running` state. You can check its logs to ensure it has successfully connected to PostgreSQL, Redis, and Local NATS.

```bash
# Get the pod name
kubectl get pods

# View logs
kubectl logs -f <auth-callout-pod-name>
```

## Step 5: Verify the Setup

Once everything is running, you can use the `nats-box` utility pod to test the entire workflow as described in previous steps:
1.  Get a shell: `kubectl run nats-box --rm -i --tty --image natsio/nats-box`
2.  Use `nats req` on `local-nats` to create a user.
3.  Use `nats pub` on the main `nats` service to test the authentication callout.

---

## Teardown

To remove all deployed resources from your cluster, run the following commands:

```bash
# Uninstall Helm releases
helm uninstall redis

# Delete resources from YAML files
kubectl delete -f ./auth-callout/
kubectl delete -f ./postgres/
kubectl delete -f ./nats-local/

# (Optional) Delete the image pull secret
kubectl delete secret regcred
```