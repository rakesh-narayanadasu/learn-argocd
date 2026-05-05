# Declarative Setup

Declarative management involves defining Kubernetes resources (such as Deployments, Services, Secrets, and ConfigMaps) and ArgoCD objects (including Applications, Repositories, and Projects) in manifest files. These files can be applied with the kubectl CLI to ensure your desired state is maintained.

Previously, we created ArgoCD applications using the CLI and UI, providing the source and destination interactively. In contrast, this approach leverages declarative manifests to define and manage these applications systematically.

>Defining your infrastructure as code in Git offers clear traceability, version control, and simplifies management of changes across different environments.

## Git Repository Structure

Assume that you have a Git repository with the structure below. The repository contains a directory called "declarative" with two subdirectories: "manifests" and "mono-app".

```text theme={null}
tree structure of Git Repository
└── declarative
    ├── manifests
    │   ├── geocentric-model
    │   │   ├── deployment.yml
    │   │   └── service.yml
    └── mono-app
        └── geocentric-app.yml
```

Inside the "mono-app" directory, you'll find an ArgoCD application YAML file. This file defines the application by specifying:

* **Project Name:** The project within ArgoCD.
* **Source Configuration:** Includes the Git repository URL, the revision (e.g., HEAD), and the path pointing to the desired Kubernetes manifests (i.e., the "geocentric-model" directory inside "declarative/manifests"). This directory contains two YAML files: one for the Deployment and one for the Service.
* **Destination Configuration:** Specifies the target cluster (using the in-cluster URL) and the namespace where resources will be deployed.
* **Sync Policy:** An optional policy that can automate the synchronization and self-healing process.

## Creating the Application

Once you have defined your manifest, create the application by running the following command:

```bash theme={null}
$ kubectl apply -f mono-app/geocentric-app.yml
```

After the application is created, ArgoCD will pull the Deployment and Service manifests from the Git repository. It will then create the resources on the target cluster either manually through a sync operation or automatically if synchronization is configured.

>By managing both your application definition and its corresponding resources in source control, you can easily track changes and maintain a consistent state across your environments.

## Example ArgoCD Application Manifest

Below is an example of an ArgoCD application manifest that encapsulates this declarative setup:

```yaml theme={null}
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: geocentric-model-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:
    repoURL: https://github.com/rakesh-narayanadasu/learn-argocd.git
    targetRevision: argocd
    path: ./declarative/manifests/geocentric-model
   
  destination:
    server: https://kubernetes.default.svc
    namespace: geocentric-model

  syncPolicy:
    syncOptions:
      - CreateNamespace=true  
    automated:
      prune: true
      selfHeal: true
```

This manifest defines the application by specifying all necessary details, including repository, revision, and resource paths. It also configures automated syncing options, ensuring that your application continuously matches the desired state defined in Git.

Using a declarative approach with Git as the single source of truth guarantees that your infrastructure changes are versioned, auditable, and easily reverted if necessary.
