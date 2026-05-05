# Deploy apps using HELM Chart

Helm is the package manager for Kubernetes that simplifies application installation and lifecycle management by using collections of YAML configuration files known as Helm Charts. These Charts bundle the YAML definitions needed for deploying a set of Kubernetes resources. With declarative application definitions as a core GitOps principle, Helm Charts can be stored in repositories specially designed for packaging and distributing them.

ArgoCD enhances this process by deploying packaged Helm Charts, monitoring them for updates, and managing their lifecycle after deployment. For example, if you have a Git repository structured with a Helm Chart, the ArgoCD CLI can be used to create an application that deploys it. By specifying the repository URL, the path to the chart, and override values via the Helm Set option, you achieve a flexible and continuous deployment process.

ArgoCD is versatile and supports deployment from various sources:

* **Artifactory Hub**
* **Bitnami Helm Charts**

Furthermore, you can deploy a Helm Chart using the ArgoCD UI. The UI simplifies repository connections by supporting SSH, HTTPS, or GitHub App integrations. It also allows you to configure repository details such as name, type, URL, and credentials for accessing private repositories.

>Once ArgoCD deploys an application using a Helm Chart, the management completely shifts to ArgoCD. Consequently, performing a `helm ls` command will not display the deployed release because it is no longer managed by Helm.

Below is an example demonstrating how to create two ArgoCD applications:

1. One deploying a Helm Chart from a Git repository.
2. Another deploying the Nginx Helm Chart from the Bitnami repository.

After deployment, the command `helm ls` confirms that the applications are managed entirely by ArgoCD.

```bash theme={null}
argocd app create random-shapes \
  --repo https://github.com/rakesh-narayanadasu/argocd-demo-app.git \
  --path helm-chart \
  --helm-set replicaCount=2 \
  --helm-set color.circle=pink \
  --helm-set color.square=violet \
  --helm-set service.type=NodePort \
  --dest-namespace default \
  --dest-server https://kubernetes.default.svc

argocd app create nginx \
  --repo https://charts.bitnami.com/bitnami \
  --helm-chart nginx \
  --revision 12.0.3 \
  --values-literal-file values.yaml \
  --dest-namespace default \
  --dest-server https://kubernetes.default.svc

helm ls
NAME   NAMESPACE REVISION UPDATED STATUS CHART APP VERSION
```

This example clearly illustrates the following:

* How to create ArgoCD applications using the CLI.
* Configuring Helm Chart parameters during deployment.
* Verifying that after deployment, the management is handled by ArgoCD rather than Helm.
