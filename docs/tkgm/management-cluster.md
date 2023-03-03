# TKG management cluster

!!! info
    The official VMware documentation is available at <https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.1/tkg-deploy-mc-21/mgmt-index.html>.

## Import node OS images

TKGm can be run either on Ubuntu or Photon VMs, whose base OS images are available on Customer Connect.

```sh
❯ vcc get versions -p vmware_tanzu_kubernetes_grid -s tkg
...
FILENAME                                                                       SIZE        BUILD NUMBER  DESCRIPTION
...
photon-3-kube-v1.24.9+vmware.1-tkg.1-f5e94dab9dfbb9988aeb94f1ffdc6e5e.ova      1.02 GB     21158328      Photon v3 Kubernetes v1.24.9 OVA
photon-3-kube-v1.23.15+vmware.1-tkg.1-2d8c52b5e3dc2b7a03bf3a04815b9474.ova     1022.45 MB  21158328      Photon v3 Kubernetes v1.23.15 OVA
photon-3-kube-v1.22.17+vmware.1-tkg.1-a10ef8e088cc2c89418bca79a2fcc594.ova     1 GB        21158328      Photon v3 Kubernetes v1.22.17 OVA
ubuntu-2004-kube-v1.24.9+vmware.1-tkg.1-b030088fe71fea7ff1ecb87a4d425c93.ova   1.78 GB     21158328      Ubuntu 2004 Kubernetes v1.24.9 OVA
ubuntu-2004-kube-v1.23.15+vmware.1-tkg.1-66940e9f3a5eda8bbd51042b1aaea6e1.ova  1.74 GB     21158328      Ubuntu 2004 Kubernetes v1.23.15 OVA
ubuntu-2004-kube-v1.22.17+vmware.1-tkg.1-df08b304658a6cf17f5e74dc0ab7543c.ova  1.77 GB     21158328      Ubuntu 2004 Kubernetes v1.22.17 OVA
...
```

You might want to select the latest version of a particular flavour, to benefit from the latest updates
and getting closer to the most recent upstream version.
In this case, we're going to pick the Photon image for Kubernetes v1.24.9

```sh
vcc download -p vmware_tanzu_kubernetes_grid -s tkg -v 2.1.0 -f 'photon-3-kube-v1.24.9+vmware.1-tkg.1-f5e94dab9dfbb9988aeb94f1ffdc6e5e.ova' --accepteula
```

and import it to our vSphere content library, to make sure it's available when needed.

```sh
govc library.import -n photon-1.24.9 ova $HOME/vcc-downloads/photon-3-kube-v1.24.9+vmware.1-tkg.1-f5e94dab9dfbb9988aeb94f1ffdc6e5e.ova
```

However, TKGm expects a VM template to be available, therefore we need to deploy a template from this content library item.
We dump the specs into a JSON file

```sh
govc import.spec $HOME/vcc-downloads/photon-3-kube-v1.24.9+vmware.1-tkg.1-f5e94dab9dfbb9988aeb94f1ffdc6e5e.ova > photon.json
```

and we edit it to make sure that

- it is marked as a template
- it does not get powered on
- the `DiskProvisioning` configuration is correct

The result looks like this

```json
{
  "DiskProvisioning": "thin",
  "IPAllocationPolicy": "dhcpPolicy",
  "IPProtocol": "IPv4",
  "NetworkMapping": [
    {
      "Name": "nic0",
      "Network": ""
    }
  ],
  "Annotation": "Cluster API vSphere image - VMware Photon OS 64-bit and Kubernetes v1.24.9+vmware.1 - https://github.com/kubernetes-sigs/cluster-api-provider-vsphere",
  "MarkAsTemplate": true,
  "PowerOn": false,
  "InjectOvfEnv": false,
  "WaitForIP": false,
  "Name": null
}
```

Finally, we can deploy the OVA into a folder dedicated to our cluster

```sh
govc folder.create /vc01/vm/tkgm
govc library.deploy -options photon.json -folder tkgm /ova/photon-1.24.9
```

## Prepare management cluster configuration

Tanzu Kubernetes Grid standalone cluster deployment can be run in unattended mode via the Tanzu CLI, feeding it with a [YAML configuration file](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.1/tkg-deploy-mc-21/mgmt-deploy-file.html).
Alternatively, [a web UI is also available](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.1/tkg-deploy-mc-21/mgmt-deploy-ui.html) for guiding the user through all the required settings.

There is actually a third option, which is preferrable if you are not familiar with or not entirely sure about all the different settings available:
it consists of using the web UI for generating the initial YAML file, and then fine tuning it, if needed, before submitting it to the Tanzu CLI.
At this point, the YAMNL descriptor can also be stored in some git repo and used as a template for future installations.

You use the Tanzu CLI to start the web UI

```sh
tanzu mc create --ui
```

which by default will listen on `127.0.0.1:8080`.
If you run it from the bastion host, you can either use the `--bind` flag or you need to create a ssh tunnel to the bastion like

```sh
ssh -l <BASTION-USER> -L 8080:localhost:8080 <BASTION-HOST-IP-OR-FQDN>
```

and then direct your browser to <http://localhost:8080>.

Now you can go through step-by-step process, providing the required information (i.e. vCenter endpoint and credentials, cluster details, ...).
In the end, you review your configuration and are able to go back to change some bits;
when everything looks good, you can go ahead and deploy the management cluster directly or export the generated YAML descriptor.

Do make sure you export the file and go through it to get yourself comfortable with all the settings.
Also, if you need to set up OIDC authentication, please take a look at the [identity management document](./identity-management.md).

## Prepare the tools

If you are planning to run TKGm in an isolated environment, you should have already [relocated all the images](./relocate-images.md),
and now you need to make the Tanzu CLI aware of your internal registry.
You can do so by setting the following variables:

- `TKG_CUSTOM_IMAGE_REPOSITORY` stores the base URL of the images, which is `<REGISTRY>/<PROJECT>`
- `TKG_CUSTOM_IMAGE_REPOSITORY_CA_CERTIFICATE` stores the base64-encoded certificate of the CA that trusts the registry's certificate

!!! warning
    The value of `TKG_CUSTOM_IMAGE_REPOSITORY_CA_CERTIFICATE` MUST NOT contain newline characters.

These values can be set either as environment variables or Tanzu CLI's variables.

!!! note ""
    === "Tanzu CLI variables (recommended)"
        You add the variables to the Tanzu CLI configuration, which is stored in a file thus persisted across different shells.
        This is the preferred method when your bastion host is used to manage only one management cluster or registry.

        ```sh
        tanzu config set env.TKG_CUSTOM_IMAGE_REPOSITORY "<REGISTRY-FQDN>/tkg"
        tanzu config set env.TKG_CUSTOM_IMAGE_REPOSITORY_CA_CERTIFICATE "$(base64 -w0 /path/to/harbor-ca.pem)"
        ```

    === "Environment variables"
        This is the preferred method if you have to deal with multiple environments with different configurations (i.e. different OCI registries),
        which is not this case.

        ```sh
        export TKG_CUSTOM_IMAGE_REPOSITORY="<REGISTRY-FQDN>/tkg"
        export TKG_CUSTOM_IMAGE_REPOSITORY_CA_CERTIFICATE="$(base64 -w0 < /path/to/harbor-ca.pem)"
        ```

Do make sure your Docker client trusts the registry as explained in the [dedicated section](./registry.md#trust-your-registry).

## Deploy the management cluster

It's finally time to build the management cluster.

You need to add one more bit to your configuration file, otherwise the Tanzu CLI will prompt you for confirmation
about deploying a management cluster on top of vSphere rather than an integrated vSphere with Tanzu (Tanzu Kubernetes Grid service).

```yaml
DEPLOY_TKG_ON_VSPHERE7: true
```

Now you can create your management cluster as

```sh
tanzu mc create --file /path/to/the/yaml/config/file -v 6
```
