# Relocate TKG images

Tanzu Kubernetes Grid images are available for download at `projects.registry.vmware.com`,
and can be copied to an internal registry for serving air-gapped or Internet-restricted installations.

The Tanzu CLI comes with an handy plugin that allows you to relocate TKGm images,
as better described in [TKGm docs](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.1/tkg-deploy-mc-21/mgmt-reqs-prep-offline.html).

First of all, you need to download all the required images to your local file system

```sh
mkdir -p tkgm-bundle && cd tkgm-bundle
tanzu isolated-cluster download-bundle \
  --source-repo projects.registry.vmware.com/tkg \
  --tkg-version v2.1.0
```

you copy the entire `tkgm-bundle` directory into your air-gapped environment[^1],
and then you can upload the images to your registry.

[^1]: if you plan to do an air-gapped TKGm installation, your bastion host does not have access to the internal registry,
therefore you need to copy the images to another host that does, which is tipically __inside__ the isolated environment

The Tanzu CLI uses `imgpkg` under the hood, so you manage authentication as described in [`imgpkg` docs](https://carvel.dev/imgpkg/docs/v0.34.0/auth/).
For example, you can use environment variables, like:

```sh
export IMGPKG_REGISTRY_HOSTNAME_0='<REGISTRY-FQDN>'
export IMGPKG_REGISTRY_USERNAME_0='<REGISTRY-USERNAME>'
export IMGPKG_REGISTRY_PASSWORD_0='<REGISTRY-PASSWORD>'
```

and then you can upload the images using the `tanzu isolated-cluster upload-bundle` command.
The username and password have been set during [Harbor preparation](registry.md#configure-users).

When using Harbor as a registry, you can download the CA certificate to a file via

```sh
curl -sSfLk -H 'accept: application/octet-stream' -o harbor-ca.pem https://<REGISTRY-FQDN>/api/v2.0/systeminfo/getcert
```

and then push the images

```sh
tanzu isolated-cluster upload-bundle \
  --source-directory tkgm-bundle \
  --destination-repo "<REGISTRY-FQDN>/tkg" \
  --ca-certificate harbor-ca.pem
```

!!! note
    Before uploading the images, make sure you create a project named `tkg`,
    as described in the [previous section](./registry.md#configure-tkg-project),
    and grant write permissions to `<REGISTRY-USERNAME>`.
