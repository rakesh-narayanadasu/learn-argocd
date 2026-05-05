# Hashicorp Vault

This explains integrating the ArgoCD Vault Plugin with HashiCorp Vault to securely fetch and inject secrets into Kubernetes resources.

We learn how the ArgoCD Vault Plugin fetches secrets from HashiCorp Vault and injects them into Kubernetes resources. This guide explains how the plugin retrieves secrets from secret management systems such as HashiCorp Vault, IBM Cloud Secrets Manager, and AWS Secrets Manager and integrates them into your Kubernetes YAML manifests.

## Overview

The ArgoCD Vault Plugin is a custom extension for ArgoCD that securely retrieves secrets from external vaults and dynamically injects them into Kubernetes configurations. In our example, HashiCorp Vault is used to store secrets securely. The plugin then retrieves these secrets and replaces placeholders in the Kubernetes manifest with the actual secret values.

HashiCorp Vault controls access to sensitive data in public or hybrid environments using secret engines. In this guide, the key-value secrets engine is enabled to store and retrieve plain text secrets. Here, the `kv put` command writes a secret specifically the `MYSQL-PASSWORD` to a defined path in Vault.

> In Vault, sensitive values stored in plain text are referenced in Kubernetes manifests using the `stringData` field rather than `data`. The `stringData` field accepts plain text without requiring Base64 encoding.

## Example Walkthrough

Below is a comprehensive example that illustrates the necessary commands and configuration details.

### Step 1: Enable the Key-Value Secrets Engine

Enable the key-value secrets engine (version 2) at a specified path:

```bash theme={null}
# Enable the key-value secrets engine at the specified path using version 2 (kv-v2)
$ vault secrets enable -path=crds kv-v2
Success! Enabled the kv secrets engine at: crds/
```

### Step 2: Write a Secret to Vault

Write the secret `MYSQL-PASSWORD` to Vault under the path `crds/mysql`:

```bash theme={null}
# Write the secret 'MYSQL-PASSWORD' to the Vault at the path 'crds/mysql'
$ vault kv put crds/mysql MYSQL-PASSWORD=1234567
Key            Value
---            -----
created_time   2022-08-31T11:17:38.755927206Z
deletion_time  n/a
destroyed      false
version        1
```

### Step 3: Prepare the Kubernetes Secret Manifest Template

Review the Kubernetes secret manifest template, which includes an annotation that maps the Vault secret path to the placeholder in the manifest:

```bash theme={null}
# Review the Kubernetes secret manifest template.
$ cat mysql-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  annotations:
    avp.kubernetes.io/path: "crds/data/mysql"
type: Opaque
stringData:
  _password: <MYSQL-PASSWORD>
```

### Step 4: Download and Install the ArgoCD Vault Plugin

Download the ArgoCD Vault Plugin binary, set the appropriate execute permissions, and move it to the local binary directory:

```bash theme={null}
# Download the ArgoCD Vault Plugin binary, set execute permissions, and move it to /usr/local/bin
$ curl -Lo argocd-vault-plugin https://github.com/argoproj-labs/argocd-vault-plugin/releases/download/v1.10.0/argocd-vault-plugin_v1.10.0_linux_amd64
$ chmod +x argocd-vault-plugin && mv argocd-vault-plugin /usr/local/bin
```

### Step 5: Configure Vault Authentication

Create a file named `vault.env` that contains the Vault configuration details. This file includes the Vault address, authentication token, and plugin-specific configuration:

```bash theme={null}
# Create a file 'vault.env' containing Vault configuration details.
$ cat vault.env
VAULT_ADDR=http://localhost:8200
VAULT_TOKEN=s.aokHnABJZD3JhABJ73nIozm9wosK02wQ
AVP_TYPE=crds
AVP_AUTH_TYPE=token
```

### Step 6: Generate the Final Kubernetes Manifest

Generate the final Kubernetes manifest with the Vault secret injected by running the `generate` command. The plugin connects to Vault using the provided configuration, retrieves the secret, and replaces the `<MYSQL-PASSWORD>` placeholder in the manifest:

```bash theme={null}
# Generate the final Kubernetes manifest with the Vault secret injected.
$ argocd-vault-plugin generate -c vault.env - < mysql-secret.yaml
```

> The annotation `avp.kubernetes.io/path: "crds/data/mysql"` in the manifest instructs the plugin to retrieve the secret key `MYSQL-PASSWORD` from the specified Vault path. Ensure your Vault configuration and paths match this specification.

## How It Works

1. **Vault Secret Storage:** The key-value secrets engine in HashiCorp Vault stores the credentials.
2. **Plugin Authentication:** The ArgoCD Vault Plugin uses the configuration provided in `vault.env` to connect and authenticate with Vault.
3. **Manifest Rendering:** The plugin reads the secret from Vault and replaces the placeholder `<MYSQL-PASSWORD>` in the Kubernetes manifest template.
4. **Deployment Integration:** The final manifest, with secrets properly injected, is ready for deployment on Kubernetes.


***

# Hashicorp Vault 2

## Overview

This setup comprises a Git repository containing Kubernetes manifests, an ArgoCD instance, a Vault server, and a Kubernetes cluster. ArgoCD periodically pulls manifests from the Git repository. One such manifest is a secret template that includes an annotation for the Vault plugin along with a placeholder for the actual secret value. For example:

```yaml theme={null}
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  annotations:
    avp.kubernetes.io/path: "crds/data/mysql"
type: Opaque
stringData:
  password: <MYSQL-PASSWORD>
```

ArgoCD uses the annotation to trigger the Vault plugin. The plugin connects to Vault and automatically fetches the required secret, transforming the manifest with the live secret data for deployment.

## Plugin Integration Approaches

ArgoCD supports custom tooling via configuration management plugins. The Vault plugin can be integrated using two approaches:

1. Direct Integration via ConfigMap\
   If the plugin is lightweight (requiring only a few lines), you can add its configuration directly to the ArgoCD ConfigMap. The repo server pod runs the plugin commands accordingly.

2. Sidecar Integration\
   For more complex plugins that may clutter the ArgoCD ConfigMap, consider deploying the plugin as a sidecar container alongside the repo server.

In this guide, we demonstrate the ConfigMap-based approach.

## Modifying the ArgoCD Repo Server

To integrate the Vault plugin, start by modifying the ArgoCD repo server deployment:

1. Define an empty directory volume to hold custom binaries.
2. Use an init container to download the ArgoCD Vault plugin binary and move it to the custom tools directory. This binary is made available to the main container during runtime.

Below is a snippet that demonstrates these changes:

```yaml theme={null}
volumes:
  - name: custom-tools
    emptyDir: {}

initContainers:
  - name: download-tools
    image: alpine:3.8
    command: ["sh", "-c"]
    args:
        - >
        wget -O argocd-vault-plugin
        https://github.com/argoproj-labs/argocd-vault-plugin/releases/download/v${AVP_VERSION}/argocd-vault-plugin_${AVP_VERSION}_linux_amd64 &&
        chmod +x argocd-vault-plugin &&
        mv argocd-vault-plugin /custom-tools/
    volumeMounts:
      - mountPath: /custom-tools
        name: custom-tools
```

After downloading the plugin, mount it in the repo server container so it is accessible in the system PATH:

```yaml theme={null}
containers:
  - name: argocd-repo-server
    volumeMounts:
      - name: custom-tools
        mountPath: /usr/local/bin/argocd-vault-plugin
```

>Ensure that the URL provided for the plugin binary is correct and that the binary version is compatible with your ArgoCD installation.

## Registering the Plugin with ArgoCD

Once the plugin binary is in place, register it with ArgoCD by updating the ConfigMap under the configuration management plugins section. After updating the ConfigMap, restart the ArgoCD repo server deployment. For instance:

```yaml theme={null}
data:
  configManagementPlugins: |-
    - name: argocd-vault-plugin
      generate:
        command: ["argocd-vault-plugin"]
        args: ["generate", "./"]
```

## Configuring Vault Connectivity

After registering the plugin, configure it to authenticate with your Vault server. You can choose between two common approaches:

1. Create a dedicated Kubernetes secret containing Vault configurations and reference it from the repo server container.
2. Embed the Vault configuration directly within each ArgoCD application’s manifest.

Both methods can be configured using the ArgoCD UI or CLI. Once the configuration is set, ArgoCD executes the plugin's generate command (with the specified arguments) to retrieve secrets from Vault and generate the final Kubernetes manifest for application deployment.

> Always ensure that Vault credentials and configurations are secured appropriately. It is recommended to use Kubernetes Secrets to store sensitive Vault information.

## Example: Verified Secret Data in Vault

After integrating and configuring the plugin, Vault stores the actual secret data. For example, a secret stored at the specified path may be retrieved as follows:

```yaml theme={null}
Read - crds/data/mysql
data: {
  "data": {
    "MYSQL-PASSWORD": "1234567"
  }
}
```

This example confirms that the Vault plugin is correctly fetching the MySQL password and making it available for Kubernetes deployments.

***

# ArgoCD Vault Plugin CLI Configuring Manully

This guide demonstrates how the ArgoCD Vault Plugin connects with HashiCorp Vault to fetch secrets and generate Kubernetes manifest files by replacing placeholders with actual secret data.

## Setting Up HashiCorp Vault

To get started, you need to deploy a Vault instance where you can add and later retrieve secrets. For this demo, we will deploy Vault using the HashiCorp Vault Helm chart and manage the deployment via ArgoCD.

### Installing Vault via Helm

Follow these steps to install Vault with Helm:

1. Add the HashiCorp Helm repository:

   ```bash theme={null}
   helm repo add hashicorp https://helm.releases.hashicorp.com
   # "hashicorp" has been added to your repositories
   ```

2. Install Vault:

   ```bash theme={null}
   helm install vault hashicorp/vault
   ```

3. Create an ArgoCD application for the Vault Helm chart. For this example, we deploy version 0.16.0 into the namespace “vault-demo.” Modify the Vault configuration to disable data storage by setting `server.datastore.enabled` to `false` and change the UI service type to `NodePort` to access the Vault UI through a browser.

   For instance, your Vault configuration snippet might resemble:

   ```bash theme={null}
   ui = true
   listener "tcp" { 
     tls_disable = 1 
     address = "[::]:8200" 
     cluster_address = "[::]:8201" 
   }
   storage "consul" { 
     path = "vault" 
     address = "HOST_IP:8500" 
   }
   ```

   And update the service configuration with:

   ```yaml theme={null}
   apiVersion: v1
   kind: Service
   metadata:
     name: vault-app
   spec:
     type: NodePort
     ports:
       - name: http
         port: 8200
         targetPort: 8200
       - name: https-internal
         port: 8201
         targetPort: 8201
     selector:
       app.kubernetes.io/instance: vault-app
       app.kubernetes.io/name: vault
       component: server
   ```

   After synchronizing the application, multiple resources will be created. The Vault pod may be in a progressing state until Vault is fully initialized.

### Accessing and Initializing Vault

After the Vault application is deployed, check the “vault-demo” namespace to verify the running resources:

```bash theme={null}
# List all resources in the vault-demo namespace
kubectl -n vault-demo get all

# Example output:
# NAME                                               READY   STATUS      RESTARTS   AGE
# pod/vault-app-0                                    0/1     Running     0          54s
# pod/vault-app-agent-injector-6947cc4648-wd9dt       1/1     Running     0          54s
#
# NAME                                               TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)              AGE
# service/vault-app                                  ClusterIP  10.110.125.148  <none>        8200/TCP,8201/TCP      54s
# service/vault-app-agent-injector-svc              ClusterIP  10.104.23.127   <none>        443/TCP              54s
# service/vault-app-internal                        ClusterIP  None            <none>        8200/TCP,8201/TCP      54s
#
# NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
# deployment.apps/vault-app-agent-injector           1/1     1            1           54s
#
# NAME                                               DESIRED   CURRENT   READY   AGE
# replicaset.apps/vault-app-agent-injector-6947cc4648  1         1         1       54s
#
# NAME                                               READY   AGE
# statefulset.apps/vault-app                         0/1     54s
```

> If the Vault service does not reflect the NodePort settings, update the service manifest manually as described above. Once updated, you can access the Vault UI using the assigned NodePort (e.g., port 31986).

- Next, initialize Vault via the UI. When initializing, choose three key shares with a threshold of at least two keys required to unseal. Save the initial root token and unseal keys securely.
- Unseal Vault by entering two of the keys in the UI
- Finally, log in using the default token authentication—the root token saved earlier. By default, only the "cubbyhole" secret engine is enabled.

## Enabling a Key-Value Secret Engine and Creating Secrets

To store application credentials, enable the KV (key-value) secret engine and create a secret:

1. In the Vault UI, enable a new secret engine.
2. Choose the key-value secret engine (version 2).
3. Set the engine path to `credentials` using default configurations.

Within the `credentials` secret engine, add a new secret under a specific path (for example, `app`) with the following fields:

* username (e.g., your name suffixed with `-vault` for demo purposes)
* password (e.g., `secure-password-vault`)
* API key (a random string)

After saving, verify that the secret is stored under the `credentials/app` path:

## Using the ArgoCD Vault Plugin

Once Vault is running, initialized, unsealed, and holds your secret data, you can use the ArgoCD Vault Plugin to access these credentials.

### Defining a Kubernetes Secret Manifest

To automatically retrieve secrets, define a Kubernetes Secret manifest with an annotation that instructs the plugin where to fetch the Vault data. For example, if your secret is stored at `credentials/app`, your manifest should be configured like this:

```yaml theme={null}
kind: Secret
apiVersion: v1
metadata:
  name: app-crds
  annotations:
    avp.kubernetes.io/path: "credentials/data/app"
type: Opaque
stringData:
  apikey: <apikey>
  username: <username>
  password: <password>
```

The plugin replaces each placeholder (the values between `<` and `>`) with the actual secret data from Vault and outputs a valid manifest, encoding values in Base64 if necessary.

### Local Installation of the ArgoCD Vault Plugin

For local testing outside of ArgoCD, follow these steps to install the ArgoCD Vault Plugin:

1. Install via Homebrew:

   ```bash theme={null}
   brew install argocd-vault-plugin
   ```

2. Alternatively, download the Linux binary from GitHub:

   ```bash theme={null}
   wget https://github.com/argoproj-labs/argocd-vault-plugin/releases/download/v1.12.0/argocd-vault-plugin_1.12.0_linux_amd64
   chmod +x argocd-vault-plugin_1.12.0_linux_amd64
   mv argocd-vault-plugin_1.12.0_linux_amd64 /usr/local/bin/argocd-vault-plugin
   ```

3. Verify the installation:

   ```bash theme={null}
   argocd-vault-plugin version
   # Expected output:
   # argocd-vault-plugin v1.12.0 (9c7288a5b2d395fea19c1100f2cd07b547cc1ee2) BuildDate: 2022-07-08T13:27:45Z
   ```

The plugin supports commands such as `generate`, `completion`, and `help`. The `generate` command is used to replace placeholder values in your secret manifest with actual data from Vault.

### Creating the Secret Manifest

Store your Kubernetes Secret manifest into a file, for example, `secret.yaml`:

```yaml theme={null}
kind: Secret
apiVersion: v1
metadata:
  name: app-crds
  annotations:
    avp.kubernetes.io/path: "credentials/data/app"
type: Opaque
stringData:
  apikey: <apikey>
  username: <username>
  password: <password>
```

> Using `stringData` here allows plain text entries, which are then encoded as needed, avoiding manual Base64 encoding.

### Configuring Vault Connection

Create an environment file (e.g., `vault.env`) that contains the necessary Vault configuration parameters:

```plaintext theme={null}
VAULT_ADDR=http://localhost:31986
VAULT_TOKEN=s.0nqTxN3rmQcoKku7DX87bWz
AVP_TYPE=vault
AVP_AUTH_TYPE=token
```

Make sure to adjust the `VAULT_ADDR` and `VAULT_TOKEN` to match your Vault instance. The `AVP_TYPE` should be set to `vault` and `AVP_AUTH_TYPE` to `token` for this demo setup.

### Generating the Manifest

Run the following command to generate the final Kubernetes manifest with secrets fetched from Vault:

```bash theme={null}
argocd-vault-plugin generate -c vault.env - < secret.yaml
```

The output will be a complete manifest with all placeholder values replaced by the actual secrets, similar to:

```yaml theme={null}
apiVersion: v1
kind: Secret
metadata:
  annotations:
    avp.kubernetes.io/path: credentials/data/app
  name: app-crds
stringData:
  apiKey: 5FGJVasdnjl-yidis67-asdkasd
  password: secure-password-vault
  username: rakesh-vault
type: Opaque
```

This manifest is now ready for deployment into your Kubernetes cluster.

***

# ArgoCD Vault Plugin Automated with ArgoCD 

In this you'll learn how to install and configure the HashiCorp Vault plugin in ArgoCD. This Vault plugin enables ArgoCD to retrieve secrets directly from HashiCorp Vault during application manifest reconciliation. We will follow the official documentation's approach using an initContainer and configuring the ArgoCD ConfigMap.

## 1. Repository Server Deployment Configuration

The first step is to modify the ArgoCD repo server deployment. This configuration uses an initContainer that downloads the Vault plugin and makes it available to the main container through a shared volume.

### Initial Deployment Example

```yaml theme={null}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
spec:
  template:
    spec:
      containers:
        - name: argocd-repo-server
          volumeMounts:
            - name: custom-tools
              mountPath: /usr/local/bin/argocd-vault-plugin
              subPath: argocd-vault-plugin
      volumes:
        - name: custom-tools
          emptyDir: {}
      initContainers:
        - name: download-tools
          image: alpine:3.8
          command: [sh, -c]
      env:
        - name: AVP_VERSION
          value: "1.12.0"
      args:
        - >
        wget -O argocd-vault-plugin
        https://github.com/argoproj-labs/argocd-vault-plugin/releases/download/v${AVP_VERSION}/argocd-vault-plugin_${AVP_VERSION}_linux_amd64 &&
        chmod +x argocd-vault-plugin &&
        mv argocd-vault-plugin /custom-tools/
```

In this deployment configuration, the ArgoCD repo server container mounts a volume named `custom-tools`. The initContainer called `download-tools` downloads the Vault plugin using `wget`, sets executable permission with `chmod +x`, and moves it to the shared volume.

### Detailed InitContainer Example

For clarity, here is an alternative snippet that highlights the initContainer setup:

```yaml theme={null}
initContainers:
  - name: download-tools
    image: alpine:3.8
    command: [sh, -c]
    env:
      - name: AVP_VERSION
        value: "1.12.0"
    args:
        - >
        wget -O argocd-vault-plugin
        https://github.com/argoproj-labs/argocd-vault-plugin/releases/download/v${AVP_VERSION}/argocd-vault-plugin_${AVP_VERSION}_linux_amd64 &&
        chmod +x argocd-vault-plugin &&
        mv argocd-vault-plugin /custom-tools/
```

This setup downloads the plugin, ensures proper permissions, and places it in `/custom-tools/`, which the repo server container will later mount.

## 2. Plugin Installation Using a Dockerfile

If you prefer embedding the plugin into a custom image, use a Dockerfile similar to the example below. This approach avoids using an initContainer by baking the Vault plugin directly into your image.

```dockerfile theme={null}
RUN apt-get update && \
    apt-get install -y \
    curl \
    awscli && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Install the AVP plugin (as root so we can copy to /usr/local/bin)
ENV AVP_VERSION=0.2.2
ENV BIN=argocd-vault-plugin
RUN curl -L -o ${BIN} https://github.com/argoproj-labs/argocd-vault-plugin/releases/download/v${AVP_VERSION}/${BIN}
RUN chmod +x ${BIN}
RUN mv ${BIN} /usr/local/bin

# Switch back to non-root user
USER 999
```

> Embedding the plugin in your custom image can simplify deployment in environments where using an initContainer is less desirable.

## 3. Configuring the Config Management Plugin

Once the Vault plugin binary is available, update the ArgoCD ConfigMap to instruct ArgoCD on how to invoke the plugin for manifest generation. Add the following configuration in your ConfigMap:

```bash
kubectl edit cm argocd-cm -n argocd
```
```yaml theme={null}
data:
  configManagementPlugins: |-
    - name: argocd-vault-plugin
      generate:
        command: ["argocd-vault-plugin"]
        args: ["generate", "./"]
```

This configuration directs ArgoCD to execute the command `argocd-vault-plugin generate ./` during the reconciliation process.

## 4. Updating the Repo Server Deployment with Vault Plugin Credentials

Below is a revised ArgoCD repo server deployment example. This configuration includes a secret reference for Vault credentials and updates the Vault plugin version to 1.7.1. Ensure that the environment variable is correctly defined as AVP\_VERSION.

```bash
kubectl edit deployment argocd-repo-server -n argocd
```
```yaml theme={null}
metadata:
  name: argocd-repo-server
spec:
  template:
    spec:
      containers:
        - name: argocd-repo-server
          volumeMounts:
            - name: custom-tools
              mountPath: /usr/local/bin/argocd-vault-plugin
              subPath: argocd-vault-plugin
          envFrom:
            - secretRef:
                name: argocd-vault-plugin-credentials   # Use this to store vault credentials vault.env as secrets
      volumes:
        - name: custom-tools
          emptyDir: {}
      initContainers:
        - name: download-tools
          image: alpine:3.8
          command: [sh, -c]
          env:
            - name: AVP_VERSION
              value: "1.12.0"
          args:
            - >
                wget -O argocd-vault-plugin
                https://github.com/argoproj-labs/argocd-vault-plugin/releases/download/v${AVP_VERSION}/argocd-vault-plugin_${AVP_VERSION}_linux_amd64 &&
                chmod +x argocd-vault-plugin &&
                mv argocd-vault-plugin /custom-tools/
          volumeMounts:
            - name: custom-tools
              mountPath: /custom-tools
      automountServiceAccountToken: true
```

After deploying these changes using, for example, `kubectl edit deployment argocd-repo-server -n argocd`, the repo server downloads the Vault plugin and processes manifests containing Vault annotations.

```bash
kubectl get pods -n argocd
```

## 5. Creating an Application Using Vault Secrets

To use the Vault plugin, enable it within your ArgoCD application. In the ArgoCD UI, create a new application (e.g., *Vault Secret App Demo*) within the default or demo project. Configure the sync policy to manual and let the target namespace be automatically created.

Within your Git repository, include a secret manifest that uses an annotation to specify the Vault path. An example manifest is:

```yaml theme={null}
kind: Secret
apiVersion: v1
metadata:
  name: app-crds
  annotations:
    avp.kubernetes.io/path: "credentials/data/app"
type: Opaque
stringData:
  apiKey: <apikey>
  username: <username>
  password: <password>
```

When the application is synchronized, the Vault plugin detects the annotation, connects to Vault (using the configuration provided either in the repo server’s secret or hard-coded), fetches the secret data from the specified path, and outputs a final Kubernetes Secret manifest.

In your ArgoCD application settings, configure the following Vault parameters (adjust based on your environment):

* AVP\_TYPE: vault
* AVP\_AUTH\_TYPE: token
* VAULT\_ADDR: e.g., [http://vault-app.vault-demo.svc.cluster.local:8200](http://vault-app.vault-demo.svc.cluster.local:8200)
* VAULT\_TOKEN: (Your Vault token)

Once configured, the application will connect to Vault and generate the desired manifest with resolved secret data.

## 6. Verifying the Application Deployment

After deploying your application, check the ArgoCD dashboard to ensure that the application status is synced and healthy.

Inspect the application details to confirm sync status and health. The Vault plugin replaces the secret placeholders with the actual data fetched from Vault.

To verify the new Kubernetes Secret with resolved data, run:

```bash theme={null}
# Verify the namespace and secret
kubectl get ns
kubectl -n <target-namespace> get secrets
# Eg:
kubectl -n vault-app get secrets app-crds 
```

To check the content of a secret, decode a value by replacing `<secret-name>` and `<key>`:

```bash theme={null}
kubectl -n <target-namespace> get secret <secret-name> -o json | jq -r '.data["<key>"]' | base64 -d

kubectl get secrets app-crds -n vault-app -o json | jq .data.username -r | base64 -d
```

## 7. Additional CLI Configuration

You can perform further adjustments using the command line. For example, edit the repo server deployment or ConfigMap with these commands:

```bash theme={null}
kubectl -n argocd edit deploy argocd-repo-server
kubectl -n argocd edit cm argocd-cm
```

After applying your changes, check the pod status:

```bash theme={null}
kubectl -n argocd get pods
```

Also, verify your Vault environment file (e.g., `vault.env`):

```bash theme={null}
cat vault.env
```

A sample `vault.env` file may look like this:

```bash theme={null}
VAULT_ADDR=http://vault-app.vault-demo.svc.cluster.local:8200
VAULT_TOKEN=s.OnqTXn3rmQoKuK7Xb87bWz
AVP_TYPE=vault
AVP_AUTH_TYPE=token
```

> Always double-check your Vault credentials and make sure that all environment variables are correctly configured to ensure secure secret management.

### The key steps included:

1. Modifying the ArgoCD repo server deployment to download the plugin via an initContainer.
2. Optionally baking the plugin into a custom Docker image.
3. Configuring the ArgoCD ConfigMap to register and invoke the plugin.
4. Creating an application with Vault annotations so that the plugin fetches the secret data from Vault.
5. Verifying the application’s deployment using both the ArgoCD dashboard and CLI tools.

By following these steps, you enhance your GitOps workflow with robust secret management through Vault integrated with ArgoCD.
