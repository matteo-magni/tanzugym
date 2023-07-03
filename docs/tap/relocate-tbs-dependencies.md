# Relocate TBS dependencies

As soon as the TAP kapp repository has been created and the packages are available,
in air-gapped environment it's mandatory to also relocate Tanzu Build Service images.

Get the version of the buildservice package

```sh
tanzu package available list buildservice.tanzu.vmware.com --namespace tap-install
```

and store it to a variable, i.e.

```sh
TBS_VERSION=1.10.10
```

Do make sure that `IMGPKG_*` and the `REGISTRY_CA_PATH` variables are set in your shell,
as explained in the [relocate TAP images section](./relocate-tap-images.md).

Download the TBS images bundle to a local tar file as

```sh
imgpkg copy -b registry.tanzu.vmware.com/tanzu-application-platform/full-tbs-deps-package-repo:${TBS_VERSION} --to-tar=tbs-full-deps.tar
```

!!! note
    If the current bastion host does not have access to the air-gapped environment,
    please move the downloaded tar file to a bastion that does and run the following commands on such host

    The file is around 12 GBs in size.
    Please make sure you have enough space on both hosts and file transfer between them is allowed.

Define the following variables

```sh
export TBS_REPOSITORY=tap/tbs-full-deps
```

You can now store the images to the registry:

```sh
imgpkg copy --tar tbs-full-deps.tar \
  --to-repo=${IMGPKG_REGISTRY_HOSTNAME_1}/${TBS_REPOSITORY} \
  --registry-ca-cert-path ${REGISTRY_CA_PATH}
```
