# Bitnami Sealed Secrets

Integrating Bitnami Sealed Secrets with ArgoCD to securely manage Kubernetes secrets in Git repositories.

We explore how Bitnami Sealed Secrets integrates with ArgoCD to securely manage Kubernetes secrets. Bitnami Sealed Secrets allows you to encrypt plain Kubernetes secrets so they can be safely stored in Git repositories public or private without exposing sensitive data. Only the Sealed Secrets controller running in your cluster can decrypt these secrets at runtime.

## Creating a Kubernetes Secret

Typically, you create a Kubernetes secret using the kubectl CLI command or by applying a YAML manifest. However, in line with GitOps best practices, all resources including secrets should be stored declaratively in Git. The challenge arises when storing Base64-encoded secrets in a repository.

For instance, you can create a Kubernetes secret from a literal value by running:

```bash theme={null}
kubectl create secret generic app-crds -o yaml --dry-run=client \
  --from-literal=username=admin-dev-group \
  --from-literal=password=paSsw0rD-1erT-diS \
  --from-literal=apikey=zaCELgL-0imfnc8mVLwwsAawjYr4Rx-Af50DDqtlx > app-crds.yaml
```

The command produces an output similar to this YAML manifest:

```yaml theme={null}
# app-crds.yaml
apiVersion: v1
data:
  apikey: emFDRUxnTC0waW1mbmM4bVZMd3dzQWF3allyNFJ4LUFmNTBERHF0bHg=
  password: cGFTc3cwckQtMWVyVC1kaVM=
  username: YWRtaW4tZGV2LWdyb3Vw
kind: Secret
metadata:
  name: app-crds
```

## Overview of Available Solutions

There are several tools for managing Kubernetes secrets securely:

* Bitnami Sealed Secrets
* HashiCorp Vault
* Kubernetes External Secrets

In this article, our focus remains on Bitnami Sealed Secrets.

## How Bitnami Sealed Secrets Work

The Sealed Secrets controller is deployed inside your Kubernetes cluster. It converts a plain Kubernetes secret into a sealed secret that is safe to store in any Git repository even a public one. Only the controller can decrypt the sealed secret, ensuring that sensitive information stays protected.

The controller can be installed in various ways, including Kustomize, Helm Charts, or directly from source. In our example, we deploy and manage the Sealed Secrets controller using ArgoCD via a Helm Chart.

> Deploying the Sealed Secrets controller via ArgoCD is optional; you can also opt to use Helm directly.

Once the controller is running, the client-side tool KubeSeal encrypts your secret using asymmetric cryptography. KubeSeal automatically retrieves the public key from the running controller. If it cannot fetch the certificate automatically, you can manually specify it using the `-cert` flag. The certificate is typically stored in the Kubernetes secret created during the controller's installation.

## Deploying the Sealed Secrets Controller with ArgoCD

To deploy the Sealed Secrets controller with ArgoCD using a Helm Chart, run the following command:

```bash theme={null}
argocd app create sealed-secrets \
  --repo https://bitnami-labs.github.io/sealed-secrets \
  --helm-chart sealed-secrets \
  --revision 2.2.0 \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace kube-system
```

The output will confirm the creation of the application:

```text theme={null}
application 'sealed-secrets' created
```

## Encrypting the Secret with KubeSeal

After deploying the controller, install the KubeSeal CLI tool. The installation command downloads and installs KubeSeal into the `/usr/local/bin` directory:

```bash theme={null}
KUBESEAL_VERSION='0.18.0' # Set this to, for example, KUBESEAL_VERSION='0.18.0'

curl -OL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION:?}/kubeseal-${KUBESEAL_VERSION:?}-linux-amd64.tar.gz"

tar -xvzf kubeseal-${KUBESEAL_VERSION:?}-linux-amd64.tar.gz kubeseal

sudo install -m 755 kubeseal /usr/local/bin/kubeseal

```

With KubeSeal installed, you can encrypt your Kubernetes secret by executing:

```bash theme={null}
# List secrets
kubectl -n kube-system get secrets

kubectl -n kube-system get secrets sealed-secrets-keyd755s -o yaml

# Extract tls.crt from the secrets sealed-secrets-keyd755s yaml ans save to sealedSecret.crt
kubectl -n kube-system get secrets sealed-secrets-keyd755s -o json | jq .data'."tls.crt"' -r | base64 -d > sealedSecret.crt

# Encrypt `app-crds.yaml` using `kubeseal`
kubeseal -o yaml --scope cluster-wide --cert sealedSecret.crt < app-crds.yaml

# Now you can push `secret.yaml` to Git repository
kubeseal -o yaml --scope cluster-wide --cert sealedSecret.crt < app-crds.yaml > secret.yaml
```

After encryption, you will have two manifest files:

1. The original Secret manifest (for reference):

   ```yaml theme={null}
    # app-crds.yaml
    apiVersion: v1
    data:
    apikey: emFDRUxnTC0waW1mbmM4bVZMd3dzQWF3allyNFJ4LUFmNTBERHF0bHg=
    password: cGFTc3cwckQtMWVyVC1kaVM=
    username: YWRtaW4tZGV2LWdyb3Vw
    kind: Secret
    metadata:
    name: app-crds
   ```

2. The SealedSecret manifest that can be stored safely in Git:

   ```yaml theme={null}
    # secret.yaml
    apiVersion: bitnami.com/v1alpha1
    kind: SealedSecret
    metadata:
    annotations:
        sealedsecrets.bitnami.com/cluster-wide: "true"
    creationTimestamp: null
    name: app-crds
    spec:
    encryptedData:
        apikey: AgBXeqD7MiFTJeaml/YAC7yuWTYT200yj/0X3DH3GyPQcr4pbBOPi8Q+V8rM83vAmHck5tmO0yJ1MylgN6QpnJGFf9uDLWrJKGjeRW829Dz2f8NVBr7+gfFSsUkk0iPoDsIDIcnef7vsxdfq4Qz2K7Euwb+TeAqYlSHLCrw3Qk2gW2dXJCiaHRy6vZ+TKwShp3WHpgYl9hdPAlxZ/Db2oNRZI6HKY1gZAuMas1tnmIPHycpqSIBNP7UV4LXx7rZ+m+18+VCByB6c+XEho06109ygIFvl3KXXzg34mzUQcY4VbJ6jP3yBBNB2TNCjwgmiH13/GhcEI153GdQNrvHH55dCx302YLDimqDWLhHmspGJUw92bySDe9Uk7ZQrss+tw8qPTxkxkWZCLhlGP6HbF4WUjQcnifOwQypsq5JQuuoIH6jvpKzlA9LagP9djZwbFMvPJjj6J+/s6GIHNx/5FuSIROdfEAAn3dYwZ26/Wc3fKCbUjLHUcnpwC8E//r/xIiNzrp4ccz/wjF1AkfE1D19vHQjkVlt7a26cFb5evfDLPJyvGBogT6Jf+NB9tpBWlO1I5STQgVTJr6n2Q61KFM4vBgBizffXidq41Zjo6BO8cpaVQTUDrwW9i16gmvWadWT7y1nVJePC5eECD6CeHc1YuPR7+hJzYosiuDnOlzOzUB/gaJOL36uXPsKQZ64W2WDqr+igP75Hbvne+48DzvcTm5t5Uav17xzhpao3ZorxokZm/fLwidAE0g==
        password: AgAkmkTL7n36om4qTVIn9lxy+04ftWcRRuDjcdviiCXYq8unpnYYg2xwQJBpliVchl6VEcI1yZ8Ii0A7oh4hvUBzthnW+k6JglCTCdP237acdYSGK+7HjPYKYpIekkJqz55s7k0+v9nbo03G0Hjg4yC9fPKDWR/yHpc7pD6IjC3bEKNHj4qbj0Ns8+YLKtJbA8j9nFaMio0BeQkzsKSB8ghIrKHMgawopTKi4Ed5CmS1zUkZf+FqHd6Lzdo+at9bbSWdIfHGumbJfEW0vYa/AJTIBcQaKpwACByAStsj6xCQHKwdQ4qkNLbrzdKXEHPQI0tpJxCbR+FzWHehJmFEyegR8zr3gFnkxSIuYyNRaVOVWEubdi70d0Z8/ECZgIcig/nVWa/3Yu9RJlghjnUEx/gJTHtPFXM8Qh4TAb/yBT0MrMpMyWcIQ64S5jZ5Hp20n+s2Aow6v3MoaFIPxb5qfBOabm1os4gqAMihDSPFyUdfByjuytQ988jHV87mFAHVAxfwoK6Mlcsb120rF1WpEZgthet83/3tt+M7C6evypqcviJHriDrSjODsMcsjTUHJ04eJux6rAOswY5sQOkYgocvYsYbBELOlzb8hWL/XS1EtL/5SQ+RrQcuhQk/aRbSK/xOl4JFI6dZ5Sjo1uogRqrwzoM/SGfc1dIzTNijk1ULkrRsu/87Y5HxA0ZAhXemzPWvNY0sQbKzG+5TfdSLYluEFA==
        username: AgCYrb7yheDvyOv+buWf3nLF3b2d6KXfnTpa2bAl2ZG7+O/CBK2b1wrZYzi1IPxN3o8e8kwsf0cUl/ChVbVaEhW5XU72haMeF30nyPGJc2ZU+JkOcgVh5MEopyqGsW2SFa7LU8e9VIE/Qck0EczG1vM3YVvy/67foOZQDHRwz+kdJKjhbuKwrvEGGldeaG0WQel4O9X2dy5H/q3hKZFpTJ+7LIzjiVOsTreEpywNNmR2yI0i/MSijlbMIgpEMVL96qw0JqF0EMW1vjUBPCxJIyplaqYnXCzmqyXIDgJ/HQN3DLbTRs9gGU1lnQjCYdHYkOIweQiGDwI/LLqoEWeDZhjU7CBQd6wlioKX8T0z5dv2hL13N3JJ2KYuvR5hiNqoPLiYkAxriYzZBdVmogBh87krxjg/SbWWnS7n4MN5/fHC9Tz902pVp6T8ZYuRhpWsTXh+E/3qGLXMWnaMF0WjxECCIqHvDJQanTbZXQBi+UJ0TcwvCs6xdZk6l5sLpJhztuq+++Z5atDkMXER2zaZ9J9qEQQmFDOyb0c4RV1HaSDVHQgRonnq1/SzPuaJIlvqsUBPnHZsLDytF9RqvHbRDkErIkTZHRE+PoUJFGiLshVK/6zq++SVB7E/iAf2ZBRV7cZDNKNbws91nCaqtBH4Jy2Uj/RTH7M7NC9MP6gVISnl32BuOSJoW9IN8guG6wtx+0okN1wNageqciB/jNuRNj0=
    template:
        data: null
        metadata:
        annotations:
            sealedsecrets.bitnami.com/cluster-wide: "true"
        creationTimestamp: null
        name: app-crds
   ```

When you apply the SealedSecret manifest to your cluster, the Sealed Secrets controller decrypts it and creates a regular Kubernetes Secret. Your pods then reference this secret as they would with any standard Kubernetes secret, with all encryption and decryption handled transparently.

By leveraging Bitnami Sealed Secrets in combination with ArgoCD and KubeSeal, you ensure that your secrets remain encrypted and secure in Git repositories while maintaining adherence to GitOps principles. This approach protects your Kubernetes clusters by providing a robust and transparent method for managing secrets.

For further information and best practices, consider visiting these resources:

* [Bitnami Sealed Secrets GitHub Repository](https://github.com/bitnami-labs/sealed-secrets)
