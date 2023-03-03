# Relocate TKG images


Tanzu Kubernetes Grid images are publicly available at `projects.registry.vmware.com/tkg`, for serving TKG installations.

The image relocation process allows you to pull those images and push them to your own OCI registry,
for easier consumption by the TKG management and workload clusters you deploy across your environments.
This is a pre-requisite for air-gapped installations, but is always highly beneficial
as it can dramatically speed up image fetching during deployments, saving time and Internet bandwidth.

The Tanzu CLI comes with a handy plugin that allows you to easily relocate TKG images,
as better described in [TKG docs](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.1/tkg-deploy-mc-21/mgmt-reqs-prep-offline.html).

First of all, you need to download all the required images to your local file system

```sh
mkdir -p tkgm-bundle && cd tkgm-bundle
tanzu isolated-cluster download-bundle \
  --source-repo projects.registry.vmware.com/tkg \
  --tkg-version v2.1.0
```

you copy the entire `tkgm-bundle` directory into your air-gapped environment[^1],
and then you can upload the images to your registry.

[^1]: if you plan to do an air-gapped TKG installation, your bastion host does not have access to the internal registry,
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

You also need the registry's CA chain certificates, to verify the endpoint when pushing images.
When using Harbor v2.x as a registry, you can download the certificates to a file via

```sh
curl -sSfLk -H 'accept: application/octet-stream' -o harbor-ca.pem https://<REGISTRY-FQDN>/api/v2.0/systeminfo/getcert
```

but if you have a different registry you can also fetch the full certificate chain via openssl:

```sh
openssl s_client -connect <REGISTRY-FQDN>:443 -servername <REGISTRY-FQDN> -showcerts </dev/null 2>/dev/null | sed -n '/-----BEGIN CERTIFICATE-----/, /-----END CERTIFICATE-----/p' > harbor-ca.pem
```

This command fetches the leaf certificate, too, and works with every SSL-enabled endpoint.
Do make sure to change the port according to your registry if it's not 443.

Then you must push the images

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

If you receive a 500 error from the registry and uploads fail, the registry might be throttling the previous command, as it runs parallel uploads.
The [troubleshooting section](./troubleshooting.md#the-registry-throttles-the-upload-bundle-command) might be helpful.
