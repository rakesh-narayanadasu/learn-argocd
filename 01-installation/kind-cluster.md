# Kind Kubernetes Installation Guide

This guide walks you through installing Kind (Kubernetes in Docker) on Linux.

## Prerequisites

- Linux system (Ubuntu/Debian-based)
- Terminal access with sudo privileges

## Installation Steps

### 1. Install Docker (Required)

Kind runs Kubernetes nodes inside Docker containers.

```bash
sudo apt update
sudo apt install -y docker.io
```

Start and enable Docker:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Add your user to the docker group (so you don't need sudo every time):

```bash
sudo usermod -aG docker $USER
```

**Important:** Log out and back in (or run `newgrp docker`) for the group changes to take effect.

### 2. Install kubectl

Download the latest stable kubectl binary:

```bash
curl -LO https://dl.k8s.io/release/$(curl -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

Verify installation:

```bash
kubectl version --client
```

### 3. Install Kind

Download and install Kind:

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x kind
sudo mv kind /usr/local/bin/kind
```

Verify installation:

```bash
kind version
```

### 4. Create a Kubernetes Cluster

Create a simple single-node cluster:

```bash
kind create cluster
```

### 5. (Optional) Custom Cluster Configuration

For a multi-node cluster, create a configuration file:

**kind-config.yaml:**
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

Then create the cluster using the config:

```bash
kind create cluster --config kind-config.yaml
```

### 6. Verify Your Cluster

Check that your cluster nodes are running:

```bash
kubectl get nodes
```

You should see your control-plane and worker nodes in a `Ready` state.

## Useful Kind Commands

- **List clusters:** `kind get clusters`
- **Delete cluster:** `kind delete cluster`
- **Delete specific cluster:** `kind delete cluster --name <cluster-name>`
- **Load Docker image into cluster:** `kind load docker-image <image-name>`

## Next Steps

Your Kind cluster is now ready! You can:
- Deploy applications using `kubectl apply -f <manifest.yaml>`
- Access the cluster using standard kubectl commands
- Test Kubernetes configurations locally before deploying to production

## Troubleshooting

- If Docker commands require sudo, ensure you've logged out and back in after adding your user to the docker group
- If kubectl can't connect, verify the cluster is running with `kind get clusters`
- Check Docker is running with `sudo systemctl status docker`
