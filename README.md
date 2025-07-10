# Kubernetes Deployment Guide

This guide provides end-to-end instructions for setting up a self-managed Kubernetes cluster on remote machines and deploying the complete application stack.

## Prerequisites

*   Three Ubuntu Linux machines with SSH access (one for the Control Plane, two for Workers).
*   A Docker Hub account.
*   `helm`(https://helm.sh/docs/intro/install/)

---

## Part 1: Remote Kubernetes Cluster Setup

This section covers the one-time setup of the Kubernetes cluster itself.

### Step 1: Prepare All Nodes (Run on Control Plane & All Workers)

The following steps must be performed on **every** machine that will be part of the cluster.

#### 1.1 SSH into Each Machine
Log in to each of your servers.

#### 1.2 Install Docker Engine
Install the Docker container runtime by following the official documentation.
**Link:** [https://docs.docker.com/engine/install/ubuntu/](https://docs.docker.com/engine/install/ubuntu/)

#### 1.3 Disable Swap
Kubernetes requires that swap be disabled.

```bash
# Check if swap is enabled
swapon --show

# If it is, turn it off
sudo swapoff -a

# To make this permanent, comment out the swap line in /etc/fstab
# Example: sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

#### 1.4 Install `cri-dockerd`
Because Docker Engine does not natively implement the Container Runtime Interface (CRI), the `cri-dockerd` service is required.

```bash
# 1. Download the .deb file
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.17/cri-dockerd_0.3.17.3-0.ubuntu-focal_amd64.deb

# 2. Install the package
sudo apt install ./cri-dockerd_0.3.17.3-0.ubuntu-focal_amd64.deb

# 3. Enable and start the cri-dockerd service
sudo systemctl daemon-reload
sudo systemctl enable --now cri-docker.service

# 4. Confirm it's running and enabled
sudo systemctl status cri-docker.socket
# The output should show 'active (running)' and 'enabled'.
```

#### 1.5 Install Kubernetes Components (`kubeadm`, `kubectl`)
First, determine the Kubernetes server version you intend to use. Then, install the matching `kubeadm`, `kubectl`, and `kubelet` packages by following the official guide for that version.

**Link:** [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

### Step 2: Initialize the Control Plane Node

These commands are run **only** on the machine you have chosen to be the Control Plane.

1.  Initialize the cluster using `kubeadm`. The `--cri-socket` flag is critical.
    ```bash
    sudo kubeadm init \
      --pod-network-cidr=192.168.0.0/16 \
      --cri-socket=unix:///var/run/cri-dockerd.sock
    ```

2.  When the command finishes, it will print a `kubeadm join` command. **Copy this entire command and save it somewhere safe.** You will need it for the worker nodes.

3.  Configure `kubectl` for your user account on the Control Plane.
    ```bash
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

### Step 3: Join the Worker Nodes

On each of your **worker nodes**, run the `kubeadm join` command that you saved from the previous step.

**Important:** You must add the `--cri-socket` flag to the join command as well.

```bash
# Example of the modified join command
sudo kubeadm join <control-plane-ip>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --cri-socket=unix:///var/run/cri-dockerd.sock
```

### Step 4: Install the CNI Network Plugin

From your **Control Plane** machine, install the Calico network plugin so pods on different nodes can communicate.

```bash
# Install the Tigera Calico operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/tigera-operator.yaml

# Install the Calico custom resource definitions
curl https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/custom-resources.yaml -O

kubectl create -f custom-resources.yaml
```

Verify that all nodes become `Ready` by running `watch kubectl get nodes`. This may take a few minutes.

### Step 5: Install the Storage Provisioner

Stateful applications like PostgreSQL and Redis require Persistent Volumes. Since this is a self-managed cluster, you must install a provisioner to create volumes using the local disk space on your nodes.

From your **Control Plane** machine, apply the `local-path-provisioner`.

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml
```

Verify the provisioner is running by checking for its pod in the `local-path-storage` namespace.

```bash
kubectl get pods -n local-path-storage
```

---

## Part 2: Application Deployment

Once the cluster is ready, you can deploy the application stack.

### Step 1: Build and Push the Multi-Architecture `auth-callout` Image

To ensure your `auth-callout` image runs on both your Mac (ARM64) and the remote servers (AMD64), you must build a multi-architecture image.

1.  **On your local Mac**, navigate to the `auth-callout` directory.
2.  Run the `docker buildx` command to build for both platforms and push to Docker Hub.
    ```bash
    docker buildx build --platform linux/amd64,linux/arm64 -t your-dockerhub-username/auth-callout:latest --push .
    ```

### Step 2: Update Deployment Manifest
Ensure `auth-callout/auth-deployment.yaml` points to your Docker Hub image and has the `imagePullPolicy: Always`.

### Step 3: Deploy All Components
From a machine with `kubectl` configured to point to your new cluster (e.g., the Control Plane), run the following commands.(Make sure to delete the already existing docker-hub-secret file that exists and create your own using the command below)

1.  **Create Docker Hub Secret**:
    ```bash
    kubectl create secret docker-registry regcred \
      --docker-server=https://index.docker.io/v1/ \
      --docker-username=your-dockerhub-username \
      --docker-password=your-docker-hub-access-token
    ```

2.  **Deploy PostgreSQL**:
    ```bash
    kubectl apply -f ./postgres/
    ```

3.  **Deploy Redis with Sentinel**:
    ```bash
    helm repo add bitnami https://charts.bitnami.com/bitnami
    helm repo update
    helm install redis bitnami/redis -f ./redis/my-redis-values.yaml
    ```

4.  **Deploy Local NATS**:
    ```bash
    kubectl apply -f ./nats-local/
    ```

5.  **Deploy the Main NATS Cluster**:
    ```bash
    helm repo add nats https://nats-io.github.io/k8s/helm/charts/
    helm repo update
    helm install nats nats/nats -f ./nats-cloud/nats-helm-values.yaml
    ```

6.  **Deploy the External NATS Load Balancer**:
    ```bash
    kubectl apply -f ./nats-cloud/nats-loadbalancer.yaml
    ```

7.  **Deploy the `auth-callout` Application**:
    ```bash
    kubectl apply -f ./auth-callout/
    ```

Your cluster is now fully deployed.

---

## Teardown

To remove all deployed applications:

```bash
helm uninstall redis nats
kubectl delete -f ./auth-callout/ -f ./postgres/ -f ./nats-local/ -f ./nats-cloud/nats-loadbalancer.yaml
kubectl delete secret regcred
```