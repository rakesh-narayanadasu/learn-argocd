# User Management RBAC ArgoCD

We explore how ArgoCD implements Role-Based Access Control (RBAC) to manage access to its resources effectively. ArgoCD uses policies defined in CSV notation, which are applied either to individual users or to SSO groups. Keep in mind that RBAC permissions for applications may differ from those applied to other resource types.

## How RBAC Works in ArgoCD

At its core, an RBAC policy in ArgoCD consists of four components:

* **Role:** A set of permissions.
* **Resource:** The target resource such as clusters, certificates, applications, repositories, or logs.
* **Action:** The operation allowed (e.g., get, create, update, delete, synchronize, or override).
* **Project/Object:** For application objects, the resource path is expressed as the project name followed by a slash and the application name.

>Custom RBAC policies allow for granular control over who can perform specific actions, ensuring that only authorized users have access to sensitive resources.

## Example: Creating a Custom "Create Cluster" Role

The following example demonstrates how to configure a custom role named `createCluster` that allows a user to create clusters in ArgoCD. In this scenario, the role is granted to a user named `jai` by patching the ArgoCD RBAC config map with the appropriate policy.

```bash theme={null}
$ kubectl -n argocd patch configmap argocd-rbac-cm \
--patch='{"data":{"policy.csv":"p, role:create-cluster, clusters, create, *, jai, role:create-cluster"}}'

configmap/argocd-rbac-cm patched
```

In this configuration, the `createCluster` role is assigned to the user `jai`, granting them the ability to create clusters. You can verify the permissions using the `argocd account can-i` command:

* Checking if `jai` can create clusters returns "yes".
* Attempting to delete a cluster returns "no" because the `createCluster` role does not include delete permissions.

## Example: Assigning a Project-Level Role

Roles can also be assigned at the project level. Consider a custom role named `kia-admins`, which grants unrestricted modification rights for any application within the `kia-project`. This role is granted to a user named `ali`. When `ali` attempts to synchronize applications within the `kia-project`, the system confirms the permissions with a "yes" response.

```bash theme={null}
$ kubectl -n argocd patch configmap argocd-rbac-cm \
--patch='{"data":{"policy.csv":"p, role:create-cluster, clusters, create, *, jai, role:create-cluster"}}'

configmap/argocd-rbac-cm patched

$ argocd account can-i create clusters '*'
yes # Logged in as - jai

$ kubectl -n argocd patch configmap argocd-rbac-cm \
--patch='{"data":{"policy.csv":"p, role:kia-admins, applications, *, kia-project/*, allow, ali, role:kia-admins"}}'
configmap/argocd-rbac-cm patched

$ argocd account can-i delete clusters '*'
no # Logged in as - jai

$ argocd account can-i sync applications kia-project/*
yes # Logged in as - ali
```

>Although user `ali` has been granted permissions within the `kia-project`, they are not authorized to synchronize applications in any other projects.

***

# ArgoCD User Management RBAC


How to manage users in ArgoCD, with a focus on local user management. By default, ArgoCD includes a built-in admin user with full super-user access. For better security practices, it is recommended that you use the admin account only for the initial configuration, then disable it once all required users have been added.

ArgoCD supports two types of user accounts:

* Local users
* Users authenticated via Single Sign-On (SSO) (for example, through Okta or similar products)

In this guide, we focus on configuring local users.

>It is best practice to disable the default admin user after setting up additional accounts to minimize security risks.

## Configuring Local Users

Local users in ArgoCD are managed by updating the ConfigMap. Each user is defined with associated capabilities, such as API key generation and UI login access. The API key capability allows a user to create a JSON Web Token (JWT) for API interactions, while the login capability grants access to the user interface.

After editing the ConfigMap, your user list might appear as shown below:

```bash theme={null}
$ argocd account list
NAME   ENABLED  CAPABILITIES
admin  true     login
jai    true     apiKey, login
ali    true     apiKey, login
```

To add or update user accounts, patch the ConfigMap with the appropriate commands:

```bash theme={null}
$ kubectl -n argocd patch configmap argocd-cm --patch='{"data":{"accounts.jai": "apiKey,login"}}'
configmap/argocd-cm patched

$ kubectl -n argocd patch configmap argocd-cm --patch='{"data":{"accounts.ali": "apiKey,login"}}'
configmap/argocd-cm patched
```

## Updating User Passwords

ArgoCD provides CLI commands to set or update user passwords. When logged in as the admin, you must enter the current admin password to change another user's password. Note that new users do not have access until their password is configured.

ArgoCD comes with two predefined roles:

* **Read-only**: Grants users access solely to view resources.
* **Admin**: Grants users full, unrestricted access.

By default, the admin account is assigned the admin role; however, you can modify this assignment or create custom roles by editing the ArgoCD RBAC ConfigMap.

For example, to update the password for the user "jai", use the following command:

```bash theme={null}
$ argocd account update-password --account jai
*** Enter password of currently logged in user (admin):
*** Enter new password for user jai:
*** Confirm new password for user jai:
Password updated
```

Alternatively, you can execute the update in a single command:

```bash theme={null}
$ argocd account update-password \
--account jai \
--new-password j€i_p@ssw0rd \
--current-password @dmin_p@$sword
Password updated
```

## Customizing Roles

The default read-only role enables users to view all resources without making modifications. To assign custom roles or modify role assignments, you must edit the ArgoCD RBAC ConfigMap. By configuring these settings, you can ensure that users without explicit role mappings are automatically granted a default read-only role.


***

# Dex Okta Connector

> This explains how to integrate DEX with Okta for authentication in ArgoCD using SAML.

How ArgoCD leverages a DEX connector to delegate authentication to an external identity service such as Okta. ArgoCD includes DEX within its installation package, enabling seamless integration with third-party identity providers (IDPs) and enhancing your authentication strategy.

**DEX** is an identity service that implements the OpenID Connect protocol to power authentication for various applications. When a user logs in via DEX, authentication is eventually validated by an external IDP. In essence, DEX acts as an intermediary between the client application (ArgoCD) and the external identity provider.

DEX supports a wide range of identity providers including Okta, Google, GitLab, GitHub, and OpenShift, as well as protocols such as SAML, OIDC, and LDAP. In this guide, we focus on configuring DEX to integrate with Okta using SAML.

## Configuring Okta for SAML

When using Okta, the following steps are required to set up a SAML application:

1. **Create a SAML Application in Okta:**\
   Provide the necessary configuration details in the Okta dashboard. For example, set the Single Sign-On (SSO) URL to your ArgoCD server URL with the `/api/dex/callback` suffix.

2. **Assign Application Access:**\
   Assign the SAML application to specific users or groups within Okta. In our example, the application is assigned to a user named Kiatim. All user and group management is handled by Okta.

3. **Obtain Integration Details:**\
   After configuration, Okta supplies an **SSO URL along with an X.509 certificate**. These values must be added to the ArgoCD ConfigMap to complete the integration with DEX.

>By default, users authenticating via Okta do not have permission to make changes within ArgoCD. To allow full operations, you must update the ArgoCD RBAC configuration.

## Updating the ArgoCD Configuration

To integrate DEX with Okta, update the ArgoCD ConfigMap with the necessary connector configuration. Use the following command and configuration snippet:

```bash theme={null}
$ kubectl -n argocd edit configmap argocd-cm
...
dex.config: |
  connectors:
    - type: saml
      id: Okta
      name: Okta
      config:
        ssoURL: <okta-idp-sso-url>
        caData: <base64encoded X.509 Certificate>
        usernameAttr: name
        emailAttr: email
        groupsAttr: groups
```

This configuration informs ArgoCD’s embedded DEX where to find your Okta instance and provides the necessary details to complete SAML authentication.

## Updating RBAC for Okta Users

After updating the DEX configuration, refresh the ArgoCD UI. You should now see a new "Login via Okta" button on the login page. To grant authenticated Okta users the permissions required to modify applications, update the ArgoCD RBAC configuration using the following steps:

```bash theme={null}
$ kubectl -n argocd edit configmap argocd-rbac-cm
...
data:
  policy.csv: |
    p, role:crudApps, applications, *, kia-project/*, allow
    g, kia-team, role:crudApps
...
configmap/argocd-rbac-cm edited
```

In the RBAC policy above, notice that we have referenced a group named "kia-team" defined in Okta. This policy applies to all users within that group, granting them full permissions to perform operations on applications within the specified ArgoCD Kubernetes project.

>Ensure that the RBAC policies are correctly configured to avoid granting excessive permissions. Regularly review your RBAC settings to maintain a secure environment.

