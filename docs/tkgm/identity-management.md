# Identity management

!!! info
    The official VMware documentation is available at <https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.1/tkg-deploy-mc-21/mgmt-iam-index.html>.

This guide focuses on OIDC authentication and, as such, leverages [Okta developer platform](https://developer.okta.com/) as an OIDC-compliant IdP.

For the moment, it takes a _ClickOps_ approach for testing things quickly, but in the future it is planned to switch to a more scripted and IaC-oriented methodology.
An [Okta Terraform provider](https://registry.terraform.io/providers/okta/okta/latest/docs) is available, indeed,
so the plan is to eventually configure everything with it, following [this blog post](https://developer.okta.com/blog/2021/11/08/k8s-api-server-oidc).

*[IdP]: Identity Provider

Pinniped is the component used for authenticating against LDAP or OIDC endpoints, and comes as a package in TKG.
It is silently installed at deploy time, but can be configured either during management cluster deployment or afterwards.

## Configure identity provider

Guides for configuring [Okta](./identity-management-okta.md) and [Azure AD](./identity-management-azuread.md) OIDC endpoints are provided as a reference,
and the configuration _might_ be adapted seamlessly to other providers, although this is not guaranteed.

You can then follow either guide to get started quickly with an OIDC-compliant identity provider,
and run some authentication and authorisation tests from your TKG platform.
The simplest use case is to have users authenticate against the IdP, retrieve the groups they're member of, and assign Kubernetes permissions based on groups.
RBAC is still configured on the Kubernetes end, whilst the authentication part is completely delegated to the IdP.

## Configure pinniped during management cluster deployment

There are a few variables you need to set during management cluster installation, either via the GUI or the flat configuration file,
as described in the section about [TKG management cluster](./management-cluster.md#prepare-management-cluster-configuration).

Such variables, as described also in the [VMware official docs](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.1/tkg-deploy-mc-21/mgmt-deploy-config-ref.html#identity-management-oidc), are

```yaml
IDENTITY_MANAGEMENT_TYPE: oidc
OIDC_IDENTITY_PROVIDER_CLIENT_ID: "$OIDC_CLIENT_ID"
OIDC_IDENTITY_PROVIDER_CLIENT_SECRET: "$OIDC_CLIENT_SECRET"
OIDC_IDENTITY_PROVIDER_GROUPS_CLAIM: "$OIDC_GROUPS_CLAIM"
OIDC_IDENTITY_PROVIDER_ISSUER_URL: "$OIDC_ISSUER_URL"
OIDC_IDENTITY_PROVIDER_NAME: "$IDP_NAME"
OIDC_IDENTITY_PROVIDER_SCOPES: "$OIDC_SCOPES"
OIDC_IDENTITY_PROVIDER_USERNAME_CLAIM: "$OIDC_USERNAME_CLAIM"
```

They must be set in the flat configuration file and the deployment process will take care of the rest, generating TLS certificates too.
For publicly trusted certificates (or at least within the enterprise, if a private CA is available) there will be a dedicated section.

!!! note
    Those variables need to be set **only** in the management cluster; in fact, all workload clusters will inherit the same configuration from their own management cluster.

## Configure pinniped after management cluster deployment

OIDC authentication can be activated on existing management clusters, too.
In fact, whether or not OIDC configs are passed to the management cluster during deployment, a `PackageInstall` resource for pinniped is created,
but its reconciliation fails until a specific secret, containing the package's configuration values, is created in the `tkg-system` namespace.
`kapp-controller` is on the lookout for such a secret and completes the package installation as soon as it is available.

To get the secret manifest generated, you need to mock a flat configuration file and fake a TKG cluster creation, in order to get back the secret's YAML definition.

Make a copy of your previous management cluster flat file and add the variables defined in the [previous paragraph](#configure-pinniped-during-management-cluster-deployment).
Set a fake value for the `VSPHERE_CONTROL_PLANE_ENDPOINT` variable, to an unused IP, like

```yaml
VSPHERE_CONTROL_PLANE_ENDPOINT: 1.1.1.1
```

otherwise the Tanzu CLI would fail during pre-flight checks, because the original endpoint is already taken by the existing cluster.

Then set a few environment variables:

```sh
export IDENTITY_MANAGEMENT_TYPE=oidc
export _TKG_CLUSTER_FORCE_ROLE="management"
export FILTER_BY_ADDON_TYPE="authentication/pinniped"
```

Then generate the secret's YAML manifest:

```sh
tanzu cluster create <CLUSTER-NAME> --dry-run --file /path/to/the/yaml/config/file > pinniped-secret.yaml
```

Make sure you pick the actual cluster name, as shown in

```sh
kubectl get clusters.cluster.x-k8s.io -n tkg-system
```

Do take a look at the generated `pinniped-secret.yaml` file, and change it if your cluster is behind a proxy.
In fact, proxy configurations do not get passed along to the pinniped values, and so you have to set the values for:

- `http_proxy`
- `https_proxy`
- `no_proxy`

The `no_proxy` variable is a comma-separated list of IP or domains patterns ([this](https://about.gitlab.com/blog/2021/01/27/we-need-to-talk-no-proxy/#no_proxy-1) is worth a read).
Remember to add to it also the CIDR range of your services network (i.e. `100.64.0.0/13`).

Then you must apply the YAML manifest to the cluster

```sh
kubectl apply -f pinniped-secret.yaml
```

You can monitor the status of the configuration looking at the resources in the `pinniped-supervisor` namespace.
In the end, you should have something like

```sh
❯ kubectl -n pinniped-supervisor get all

NAME                                                   READY   STATUS      RESTARTS   AGE
pod/pinniped-post-deploy-controller-64c59657c5-hsd5w   1/1     Running     0          109m
pod/pinniped-post-deploy-job-scstf                     0/1     Completed   0          59m
pod/pinniped-supervisor-77d67dc647-f7bpd               1/1     Running     0          59m
pod/pinniped-supervisor-77d67dc647-t8fqq               1/1     Running     0          59m

NAME                          TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
service/pinniped-supervisor   NodePort   100.69.192.203   <none>        443:31234/TCP   109m

NAME                                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/pinniped-post-deploy-controller   1/1     1            1           109m
deployment.apps/pinniped-supervisor               2/2     2            2           109m

NAME                                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/pinniped-post-deploy-controller-64c59657c5   1         1         1       109m
replicaset.apps/pinniped-supervisor-77d67dc647               2         2         2       109m

NAME                                 COMPLETIONS   DURATION   AGE
job.batch/pinniped-post-deploy-job   1/1           8s         59m
```

It is worth to check the `pinniped-concierge` namespace as well:

```sh
❯ kubectl -n pinniped-concierge get all

NAME                                                     READY   STATUS    RESTARTS   AGE
pod/pinniped-concierge-5f85876ff6-fpq9t                  1/1     Running   0          111m
pod/pinniped-concierge-5f85876ff6-n497d                  1/1     Running   0          111m
pod/pinniped-concierge-kube-cert-agent-f6fbff54f-7jtfp   1/1     Running   0          110m

NAME                               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/pinniped-concierge-api     ClusterIP   100.71.104.69   <none>        443/TCP   111m
service/pinniped-concierge-proxy   ClusterIP   100.71.213.71   <none>        443/TCP   111m

NAME                                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/pinniped-concierge                   2/2     2            2           111m
deployment.apps/pinniped-concierge-kube-cert-agent   1/1     1            1           110m

NAME                                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/pinniped-concierge-5f85876ff6                  2         2         2       111m
replicaset.apps/pinniped-concierge-kube-cert-agent-f6fbff54f   1         1         1       110m
```

!!! info
    Using Kube-VIP, the only way to expose pinniped is via a NodePort service,
    thus do remember to set the appropriate rules in your network firewalls to allow the traffic to port 31234/tcp.

## Provide own TLS certificates

<https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.1/tkg-deploy-mc-21/mgmt-iam-custom-pinniped-certificates.html>

## Configure redirect URI

Now that you have pinniped configured and running, you can go back to your IdP configuration portal and set the redirect URI for your application.
Edit the `General Settings` section of your application and set the sign-in redirect URI to `https://<YOUR-MGMT-CLUSTER-ENDPOINT>:31234/callback`.
Use the very same endpoint you set in your certificate, if you happened to change it.

## Get the kubeconfig file

Use the Tanzu CLI to retrieve the kubeconfig file to authenticate via OIDC:

```sh
tanzu mc kubeconfig get --export-file /tmp/tkgm.oidc.kubeconfig
```

Open the file and look at the `users` object; the `tanzu pinniped-auth login` command is used to authenticate the user, with a few configuration options:

- make sure the `--issuer` flag points to the same endpoint as the [redirect URI](#configure-redirect-uri) (without the trailing `/callback`);
- add the item `--skip-browser` to the end of the `args` list, to prevent the Tanzu CLI from trying to run a browser to let the user authenticate
(on SSH-based jumphosts you do not have nor you do want to have a GUI).

Run a `kubectl get` command with the new kubeconfig file to get prompted for authentication

```sh
❯ kubectl get pods --kubeconfig ~/t1xl/t1xl.oidc.kubeconfig
Log in by visiting this link:

    https://my.cluster.endpoint:31234/oauth2/authorize?access_type=offline&client_id=pinniped-cli&code_challenge=lGjmpPEfyN0544UEOr2QYWawsWBzKOorlQG-Ws73FHs&code_challenge_method=S256&nonce=48cd4fc34b4e789ee8e6e590ae54aab0&redirect_uri=http%3A%2F%2F127.0.0.1%3A45359%2Fcallback&response_mode=form_post&response_type=code&scope=offline_access+openid+pinniped%3Arequest-audience&state=93b02cc91ad38b8e905844abcf0c6144

    Optionally, paste your authorization code:
```

Paste the URL in your browser, authenticate to Okta and you get a code back that you have to paste into the console: you are now authenticated.
You can see your token details (including the groups that have been passed along) at `~/.config/tanzu/pinniped/sessions.yaml`.

Your request is going to fail unless you have already [configured RBAC](#configure-rbac) on your cluster.

## Configure RBAC

If you have correctly configured [users and groups](#create-users-and-groups) and the [authorization server](#configure-authorization-server),
you should now find a list of groups in the `~/.config/tanzu/pinniped/sessions.yaml` file for your TKG ID token.

The authorization part still lies on the Kubernetes part, leveraging `Role`s and `ClusterRole`s for defining permissions,
and `RoleBinding`s and `ClusterRoleBinding`s for granting them to either users or groups.

TKG comes with a few pre-defined `ClusterRole`s

- cluster-admin: Full access to the cluster. When used in a RoleBinding, this role gives full access to any resource in the namespace specified in the binding.
- admin: Admin access to most resources in a namespace. Can create and modify roles and role bindings within the namespace.
- edit: Read-write access to most objects in a namespace, such as deployments, services, and pods. Cannot view, create, or modify roles and role bindings.
- view: Read-only access to most objects in a namespace. Cannot view, create, or modify roles and role bindings

More info at [VMware docs](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.1/tkg-deploy-mc-21/mgmt-iam-configure-rbac.html).

As an example, a simple `ClusterRoleBinding` that grants a group `view` permissions on all namespaces can be defined as:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dev-view
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: dev
```
