# TKG workload cluster

!!! info
    The official VMware documentation is available at <https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.1/using-tkg-21/workload-index.html>.

Tanzu Kubernetes Grid workload clusters are the Kubernetes clusters responsible for running your applications,
and can be created using the Tanzu CLI from a TKG management cluster on the same platform (i.e. vSphere).

You can create three types of workload clusters:

- Class-based clusters (default)
- Plan-based clusters (legacy)
- TKC-based clusters (legacy)

More information about the different classes are available on [VMware docs website](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2/about-tkg/clusters.html#types).
This guide explains how to create class-based clusters.

## Prepare workload cluster definition

The class-based workflow cluster manifest can be created from a flat file, similar to the one used for building the management cluster.
Following [VMware docs](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.1/using-tkg-21/workload-clusters-configure-vsphere.html),
you can create the flat configuration file, for example:

```yaml
AVI_CONTROL_PLANE_HA_PROVIDER: false
CLUSTER_ANNOTATIONS: 'description:alpha-workload-cluster,location:non,mc:t1xl'
CLUSTER_CIDR: 100.96.0.0/11
SERVICE_CIDR: 100.64.0.0/13
CLUSTER_PLAN: dev
NAMESPACE: default
CNI: antrea
ENABLE_AUDIT_LOGGING: false
ENABLE_CEIP_PARTICIPATION: false
ENABLE_MHC: true
ENABLE_MHC_CONTROL_PLANE: true
ENABLE_MHC_WORKER_NODE: true
MHC_UNKNOWN_STATUS_TIMEOUT: 5m
MHC_FALSE_STATUS_TIMEOUT: 12m
IDENTITY_MANAGEMENT_TYPE: none
INFRASTRUCTURE_PROVIDER: vsphere
OS_ARCH: amd64
OS_NAME: photon
OS_VERSION: "3"
TKG_HTTP_PROXY_ENABLED: false
VSPHERE_CONTROL_PLANE_ENDPOINT: my.endpoint.fqdn.or.ip
VSPHERE_DATACENTER: /vc01
VSPHERE_DATASTORE: /vc01/datastore/vsanDatastore
VSPHERE_FOLDER: /vc01/vm/tkgm/alpha
VSPHERE_INSECURE: false
VSPHERE_NETWORK: /vc01/network/tkgm_vip_workload
VSPHERE_PASSWORD: <encoded:bXktb3duLXBhc3N3b3JkCg==>
VSPHERE_RESOURCE_POOL: /vc01/host/vc01cl01/Resources
VSPHERE_SERVER: my.vcenter.fqdn.or.ip
VSPHERE_SSH_AUTHORIZED_KEY: ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICAKfCTddh2+U/Y3DmhTwq9pqr+bWCZFxEOF1S2LVxsR
VSPHERE_TLS_THUMBPRINT: B6:C1:C2:77:2F:E5:BA:C7:C1:C3:AD:59:E1:76:E4:19:13:75:30:ED
VSPHERE_USERNAME: administrator@vsphere.local
```

You can also use a different user with fewer privileges than `administrator@vsphere.local` for building the workload cluster,
more info on this will be provided in a different section.
All settings are detailed in the [VMware docs website](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.1/tkg-deploy-mc-21/mgmt-deploy-file.html).

You can get the vCenter SHA1 fingerprint (for the `VSPHERE_TLS_THUMBPRINT` setting) as

```sh
openssl s_client -connect my.vcenter.fqdn.or.ip:443 -servername my.vcenter.fqdn.or.ip -showcerts </dev/null 2>/dev/null | openssl x509 -fingerprint -noout -sha1 | cut -d= -f2
```

!!! warning
    The `-sha1` flag is necessary especially on MacOS's openssl, where the default output digest is md5.

You may now convert the flat file to the class-based file.
First of all, set the Tanzu CLI to not apply automatically the generated class-based file:

```sh
tanzu config set features.cluster.auto-apply-generated-clusterclass-based-configuration false
```

then you can produce such file as

```sh
tanzu cluster create alpha --file alpha-config.yaml --dry-run > alpha-spec.yaml
```

This command takes also care of validating your inputs, therefore if something is not right (i.e. the designated vSphere folder does not exist) it will fail with a meaningful message.

The generated YAML file defines these resource types:

- `VSphereCPIConfig`
- `VSphereCSIConfig`
- `ClusterBootstrap`
- `Secret`
- `Cluster`

The `ClusterBootstrap` defines which components will be deployed to the new cluster as Tanzu packages.
One thing to notice is that those components do not include the version, but they rather specify `*`

```yaml
apiVersion: run.tanzu.vmware.com/v1alpha3
kind: ClusterBootstrap
metadata:
  annotations:
    tkg.tanzu.vmware.com/add-missing-fields-from-tkr: v1.24.9---vmware.1-tkg.1
  name: alpha
  namespace: default
spec:
  additionalPackages:
  - refName: metrics-server*
  - refName: secretgen-controller*
  - refName: pinniped*
  cpi:
    refName: vsphere-cpi*
    valuesFrom:
      providerRef:
        apiGroup: cpi.tanzu.vmware.com
        kind: VSphereCPIConfig
        name: alpha
  csi:
    refName: vsphere-csi*
    valuesFrom:
      providerRef:
        apiGroup: csi.tanzu.vmware.com
        kind: VSphereCSIConfig
        name: alpha
  kapp:
    refName: kapp-controller*
```

to include any possible version that may have been made available to the cluster via a Tanzu package repository.

The downside is that this resource cannot be used as a desired state and thus included into a GitOps-like process,
as those `*`s are immediately translated into actual versions as soon as the workload cluster gets created.

## Create workload cluster

You now have the YAML manifest for the new cluster, which must be applied to the management cluster.

Make sure you do have your management cluster configuration and set it as your default kubeconfig file.
You should be able to find it at `~/.kube-tkg/config`, so you just have to set

```sh
export KUBECONFIG=~/.kube-tkg/config
```

and you can check you're connected to the right endpoint via

```sh
kubectl get nodes
```

You can also retrieve the kubeconfig file via

```sh
tanzu mc kubeconfig get --admin
```

and switch to the right context.
Now apply the workload cluster manifest:

```sh
kubectl apply -f alpha-spec.yaml
```

You can monitor the status of the resources running a `kubectl get` wrapped in a `watch`:

```sh
watch kubectl get md,ma,vspherevm,vspheremachines
```

and

```sh
❯ tanzu cluster list

  NAME   NAMESPACE  STATUS   CONTROLPLANE  WORKERS  KUBERNETES        ROLES   PLAN  TKR
  alpha  default    running  1/1           2/2      v1.24.9+vmware.1  <none>  dev   v1.24.9---vmware.1-tkg.1
```

## Connect to the new cluster

Ask the Tanzu CLI to fetch the kubeconfig file for you and store it to a file.
To get the configuration added to the currently active kubeconfig file you must omit the `--export-file` flag:

```sh
❯ tanzu cluster kubeconfig get alpha --admin --export-file alpha.kubeconfig

Credentials of cluster 'alpha' have been saved
You can now access the cluster by running 'kubectl config use-context alpha-admin@alpha' under path 'alpha.kubeconfig'
```

Check the nodes status

```sh
❯ export KUBECONFIG=$(pwd)/alpha.kubeconfig
❯ kubectl get nodes
NAME                                STATUS   ROLES           AGE     VERSION
alpha-md-0-9qpjk-8694659b79-7mpxs   Ready    <none>          91s     v1.24.9+vmware.1
alpha-md-0-9qpjk-8694659b79-vfrbk   Ready    <none>          89s     v1.24.9+vmware.1
alpha-rv6pp-fkqg4                   Ready    control-plane   2m28s   v1.24.9+vmware.1
```

You can now test the cluster with a sample nginx web server.
As this environment is isolated, you need to push the image to the internal registry first (this step might be documented elsewhere in the future),
and then you can run it.
In this example, I created an `apps` project on the registry and pushed a `nginx:stable-alpine` to it.

```sh
kubectl run --image registry.my.domain/apps/nginx:stable-alpine nginx
kubectl port-forward pods/nginx 8080:80
```

and, in another terminal,

```sh
curl localhost:8080
```

the response confirms the pod is running correctly.

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## Configure Tanzu standard repository

TKG provides a number of packages available to be installed and used right out of the box,
based on the Carvel kapp packaging system.
They are delivered as part of a kapp repository, which is also available as OCI image,
thats has been relocated in the internal registry during [image relocation](./relocate-images.md).

You can add the repository as

```sh
tanzu package repository add tanzu-standard --url ${TKG_CUSTOM_IMAGE_REPOSITORY}/packages/standard/repo:v2.1.0 --namespace tkg-system
```

and, as soon as the reconciliation succeeds, you can list available packages:

```sh
❯ tanzu package available list

  NAME                                          DISPLAY-NAME
  cert-manager.tanzu.vmware.com                 cert-manager
  contour.tanzu.vmware.com                      contour
  external-dns.tanzu.vmware.com                 external-dns
  fluent-bit.tanzu.vmware.com                   fluent-bit
  fluxcd-helm-controller.tanzu.vmware.com       Flux Helm Controller
  fluxcd-kustomize-controller.tanzu.vmware.com  Flux Kustomize Controller
  fluxcd-source-controller.tanzu.vmware.com     Flux Source Controller
  grafana.tanzu.vmware.com                      grafana
  harbor.tanzu.vmware.com                       harbor
  multus-cni.tanzu.vmware.com                   multus-cni
  prometheus.tanzu.vmware.com                   prometheus
  whereabouts.tanzu.vmware.com                  whereabouts
```

The packages are available in all namespaces, as the repository has been installed in
kapp-controller's packaging global namespace.
