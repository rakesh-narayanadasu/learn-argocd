# Installation Options

ArgoCD installation models, including core and multi-tenant options, along with installation commands and configurations for different environments.

ArgoCD can be installed in two primary modes: a core installation for single-tenant use and a multi-tenant installation for environments requiring isolated access for multiple teams.

## Core Installation

The core installation is designed for users running ArgoCD as a standalone service. This mode provides a minimal, non-high availability (non-HA) deployment, making it perfect for simpler setups that do not require multi-tenancy.

## Multi-Tenant Installation

Multi-tenant deployments are ideal for organizations with multiple application development teams, typically managed by a centralized platform team. Within the multi-tenant model, you have two installation variants:

### 1. Non-High Availability (non-HA)

This variant is excellent for evaluation, testing, and proof-of-concept deployments, even though it is not recommended for production use.

* **install.yaml:**\
  Deploys ArgoCD with cluster-admin access, making it suitable for clusters where ArgoCD also deploys applications. Additionally, the provided credentials allow for deploying to remote clusters.

* **namespace-installed.yaml:**\
  Configures ArgoCD for namespace-level access, offering restricted permissions. This option is useful when you want to limit ArgoCD’s access while still deploying applications in the same cluster if needed.

### 2. High Availability (HA)

For production environments, the high availability option is the recommended choice. It improves resilience by deploying multiple replicas for critical components. Two manifests are provided:

* **ha-install.yaml**
* **ha-namespace-installed.yaml**


## Installing ArgoCD with Helm

In addition to the standard installation, ArgoCD can be installed using Helm through a community-maintained chart. By default, the Helm chart deploys the non-HA version of ArgoCD.

After installation, download the ArgoCD CLI from its GitHub repository and move it to your local binary directory. The CLI allows you to efficiently interact with the ArgoCD API server.

## Installation Commands

Use the following commands to install ArgoCD:

## Installing ArgoCD

Start by creating the "argocd" namespace and applying the stable manifest:

```bash theme={null}
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests
```

We are using version 2.4.11 of ArgoCD. If you need to install a specific version, run:

```bash theme={null}
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.12/manifests/install.yaml
```

You can also install the ArgoCD CLI on your machine. For Homebrew users, execute:

```bash theme={null}
brew install argocd
```

After installing the CLI, patch the ArgoCD server service to expose it externally as a LoadBalancer and then forward the port:

```bash theme={null}
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

```bash theme={null}
kubectl port-forward svc/argocd-server -n argocd 8800:443

kubectl port-forward svc/argocd-server -n argocd --address 0.0.0.0 8800:443
```

```bash
curl -k https://localhost:8800

https://98.86.226.160:8800/
```

This installation is performed on a single-node Kubernetes cluster running version 1.24.3. Verify your node status with:

```bash theme={null}
kubectl get nodes
```

Expected output:

```plaintext theme={null}
NAME            STATUS   ROLES         AGE   VERSION
demo-cluster    Ready    control-plane 18h   v1.24.3
```

## Verifying the Installation

After installation, check the resources created in the "argocd" namespace. The following command displays all deployments, pods, and services:

```bash theme={null}
kubectl get all -n argocd
```

To check the status of services:

```bash theme={null}
kubectl get svc -n argocd
```

## Exposing the ArgoCD Server

By default, ArgoCD services are configured with the ClusterIP type, which restricts external access. To access the ArgoCD UI, modify the ArgoCD server service to use the NodePort type. Edit the service and change the "type" field:

```yaml theme={null}
creationTimestamp: "2022-09-23T14:01:00Z"
labels:
  app.kubernetes.io/component: server
  app.kubernetes.io/name: argocd-server
  app.kubernetes.io/part-of: argocd
name: argocd-server
namespace: argocd
resourceVersion: "81300"
uid: 23c477a6-23ad-4c14-8874-2f144ba396e3
spec:
  clusterIP: 10.98.110.228
  clusterIPs:
  - 10.98.110.228
  internalTrafficPolicy: Cluster
  ipFamiles:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
  - name: https
    port: 443
    protocol: TCP
    targetPort: 8080
  selector:
    app.kubernetes.io/name: argocd-server
  sessionAffinity: None
  type: ClusterIP
  loadBalancer: {}
```

After editing, verify the change by listing the services:

```bash theme={null}
kubectl get svc -n argocd
```

You should now see the ArgoCD server service type as NodePort:

```plaintext theme={null}
NAME                              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
argocd-applicationset-controller   ClusterIP   10.100.58.34   <none>        7000/TCP,8080/TCP            105s
argocd-dex-server                  ClusterIP   10.109.179.192 <none>        5556/TCP,5557/TCP,5558/TCP   105s
argocd-metrics                     ClusterIP   10.100.111.162 <none>        8082/TCP                    104s
argocd-notifications-controller-metrics ClusterIP 10.110.116.143 <none>        9001/TCP                    104s
argocd-redis                       ClusterIP   10.106.239.177 <none>        6379/TCP                    104s
argocd-repo-server                 ClusterIP   10.101.4.27    <none>        8081/TCP,8084/TCP           104s
argocd-server                      NodePort    10.98.110.228  <none>        80:30663/TCP,443:31194/TCP   104s
argocd-server-metrics              ClusterIP   10.97.180.219  <none>        8083/TCP                    104s
```

With the NodePort configuration, the ArgoCD server is now accessible externally via the server’s IP address and its designated NodePort (for example, 30663).

Open your web browser and navigate to the server’s IP (such as 139.59.21.103) along with the NodePort. Note that because the server uses a self-signed certificate, your browser will display a warning regarding the connection's privacy.

## Logging into the ArgoCD UI

The default login credentials for ArgoCD are:

* Username: admin
* Password: Retrieved from the initial admin secret

To retrieve the initial admin password, inspect the secret within the "argocd" namespace. First, list the secrets:

```bash theme={null}
kubectl get secret -n argocd
```

Expected output:

```plaintext theme={null}
NAME                           TYPE     DATA   AGE
argocd-initial-admin-secret    Opaque   1      2m44s
argocd-notifications-secret    Opaque   0      3m4s
argocd-secret                  Opaque   5      3m4s
```

Next, retrieve the secret in JSON format:

```bash theme={null}
kubectl get secret argocd-initial-admin-secret -n argocd -o json
```

To decode the password, run:

```bash theme={null}
kubectl get secret argocd-initial-admin-secret -n argocd -o json | jq .data.password -r
```

Then decode the base64 output:

```bash theme={null}
kubectl get secret argocd-initial-admin-secret -n argocd -o json | jq .data.password -r | base64 -d
```

Copy the decoded password and use it with the username "admin" to log in. Once logged in, update your password via the UI by visiting the user settings page, where you can configure repositories, certificates, clusters, projects, and accounts.

To update your password, enter the current password, specify your new password, and confirm the new password

After updating, the UI will automatically log you out. Log back in using your new credentials.

## Installing the ArgoCD CLI

Managing ArgoCD from the command line is facilitated by the ArgoCD CLI. Download the appropriate CLI binary for your system from the releases page. For version 2.4.11 on Linux AMD64, run:

```bash theme={null}
wget https://github.com/argoproj/argo-cd/releases/download/v2.4.11/argocd-linux-amd64
```

After downloading, rename the file, make it executable, and move it to your system's binary path:

```bash theme={null}
mv argocd-linux-amd64 argocd
chmod +x argocd
mv argocd /usr/local/bin/
```

Test the CLI by checking its available commands:

```bash theme={null}
argocd
```

## Logging into ArgoCD via CLI

Access the ArgoCD server using the CLI by executing the login command with the server’s IP address:

```bash theme={null}
argocd login 10.98.110.228

argocd login localhost:8800 --insecure
```

Since the server uses a self-signed certificate, you will be prompted with a certificate warning:

```plaintext theme={null}
WARNING: server certificate had error: x509: cannot validate certificate for 10.98.110.228 because it doesn't contain any IP SANs. Proceed insecurely (y/n)? y
Username: admin
Password:
```

Once logged in, you can list applications and clusters.

To list applications:

```bash theme={null}
argocd app list
```

To display available clusters, run:

```bash theme={null}
argocd cluster list
```

By default, the Kubernetes cluster on which ArgoCD is installed becomes the target cluster. In future lessons, we will explore how to deploy applications across multiple clusters.

## Summary of Commands and Resource Checks

Below is a summary of the most important commands executed during this installation process:

| Command                      | Description                                                                 |
| ---------------------------- | --------------------------------------------------------------------------- |
| `kubectl get svc -n argocd`  | Displays the ArgoCD service configuration in the argocd namespace.          |
| `argocd login 10.98.110.228` | Logs into the ArgoCD server via the CLI.                                    |
| `argocd app list`            | Lists the deployed applications (empty initially).                          |
| `argocd cluster list`        | Lists the available clusters, showing the default in-cluster configuration. |
