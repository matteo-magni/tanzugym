# Relocate TAP images

For air-gapped environments you need to relocate TAP images to an internal registry from Tanzu Network.

First of all, you need an account on Tanzu Network and define a few variables for authenticating to it:

```sh
export IMGPKG_REGISTRY_HOSTNAME_0='registry.tanzu.vmware.com'
export IMGPKG_REGISTRY_USERNAME_0='<TANZUNET-USERNAME>'
export IMGPKG_REGISTRY_PASSWORD_0='<TANZUNET-PASSWORD>'
```

Define the `TAP_VERSION` variable, i.e.

```sh
TAP_VERSION='1.5.2'
```

You can list the available TAP versions either by logging into Tanzu Network or using [pivnet](https://github.com/pivotal-cf/pivnet-cli):

```sh
pivnet rs --product-slug='tanzu-application-platform' --format yaml | yq 'map(select(.availability == "All Users")|.version)|sort_by(.)'
```

Download the TAP images bundle to a local tar file as

```sh
imgpkg copy \
  -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:${TAP_VERSION} \
  --to-tar tap-packages.tar \
  --include-non-distributable-layers
```

!!! note
    If the current bastion host does not have access to the air-gapped environment,
    please move the downloaded tar file to a bastion that does and run the following commands on such host

    The file is around 9 GBs in size.
    Please make sure you have enough space on both hosts and file transfer between them is allowed.

Define the following variables

```sh
export IMGPKG_REGISTRY_HOSTNAME_1='<REGISTRY-FQDN>'
export IMGPKG_REGISTRY_USERNAME_1='<REGISTRY-USERNAME>'
export IMGPKG_REGISTRY_PASSWORD_1='<REGISTRY-PASSWORD>'

export TAP_REPOSITORY=tap/tap-packages
export REGISTRY_CA_PATH=/path/to/registry-ca.pem
```

If using Harbor, make sure you have prepared a project to host TAP images, i.e. `tap`,
and build the destination repository string accordingly.

!!! tip "Create a Harbor project via API"
    You can create a new Harbor project named `tap` with the following command, assuming the user you have set previously has admin rights:

    ```sh
    curl -sSfL -X POST -u "${IMGPKG_REGISTRY_USERNAME_1}:${IMGPKG_REGISTRY_PASSWORD_1}" -d '{"project_name": "tap"}' -H "Content-Type: application/json" -H "Accept: application/json" https://${IMGPKG_REGISTRY_HOSTNAME_1}/api/v2.0/projects
    ```

    Full Harbor API documentation is available at your own Harbor installation at `https://${IMGPKG_REGISTRY_HOSTNAME_1}/devcenter-api-2.0`

You can now store the images to the registry:

```sh
imgpkg copy \
  --tar tap-packages.tar \
  --to-repo ${IMGPKG_REGISTRY_HOSTNAME_1}/${TAP_REPOSITORY} \
  --include-non-distributable-layers \
  --registry-ca-cert-path ${REGISTRY_CA_PATH}
```
