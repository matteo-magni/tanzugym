# Deployment and configuration

A very common scenario for customers adopting Tanzu Kubernetes Grid platform is running it within an isolated environment,
be it either just Internet-restricted or even air-gapped.
To be able to tackle such situations, it is necessary to fulfil a few tasks and meet some prerequisites.

Tanzu Kubernetes Grid images, as well as applications', must be available from within the isolated environment, thus it's necessary to fetch
and make them available in advance to the platform via an internal OCI registry.
VMware supports CNCF's [Harbor](https://goharbor.io/), and, since TKGM 2.1.0, provides a ready-to-use OVA template,
downloadable from [Customer Connect](https://customerconnect.vmware.com/).

## Download template

After you've downloaded `vcc` and authenticated as described in the [bastion host preparation guide](./bastion.md#vcc),
you can fetch the Harbor OVA:

First of all, list the files available for tkg 2.1.0

```sh
vcc get files -p vmware_tanzu_kubernetes_grid -s tkg -v 2.1.0
```

and find the Harbor OVA file:

```text
FILENAME                                                                       SIZE        BUILD NUMBER  DESCRIPTION
...
photon-4-harbor-v2.6.3+vmware.1-9c5c48c408fac6cef43c4752780c4b048e42d562.ova   1.27 GB                   Photon v4 Harbor v2.6.3 OVA
...
```

Save the name to a variable, for DRY purposes:

```sh
HARBOR_OVA="photon-4-harbor-v2.6.3+vmware.1-9c5c48c408fac6cef43c4752780c4b048e42d562.ova"
```

Then download the file:

```sh
vcc download -p vmware_tanzu_kubernetes_grid -s tkg -v 2.1.0 -f ${HARBOR_OVA} --accepteula
```

The file will be stored by default at `$HOME/vcc-downloads/`.

## Install appliance

You can now deploy a VM out of the downloaded OVA using `govc`.
Make sure it is [installed](./bastion.md#govc) and authentication has been properly configured, like:

```sh
export GOVC_URL=my.vcenter.fqdn
export GOVC_USERNAME=my.vcenter.username
export GOVC_PASSWORD=my.vcenter.password
export GOVC_INSECURE=true
```

The last line is needed if the vCenter certificate is not yet trusted by your bastion host.

You need a configuration file for customising the template to your needs and produce an actual VM.
Please export the spec to a file as

```sh
govc import.spec $HOME/vcc-downloads/${HARBOR_OVA} | jq > harbor.json
```

and you get something like the following sample JSON snippet

??? example "harbor.json"
    ```json
    {
      "DiskProvisioning": "flat",
      "IPAllocationPolicy": "dhcpPolicy",
      "IPProtocol": "IPv4",
      "PropertyMapping": [{
          "Key": "guestinfo.root_password",
          "Value": ""
        },
        {
          "Key": "guestinfo.allow_root_ssh",
          "Value": "True"
        },
        {
          "Key": "guestinfo.harbor_hostname",
          "Value": ""
        },
        {
          "Key": "guestinfo.harbor_admin_password",
          "Value": ""
        },
        {
          "Key": "guestinfo.harbor_database_password",
          "Value": ""
        },
        {
          "Key": "guestinfo.harbor_scanner_enable",
          "Value": ""
        },
        {
          "Key": "guestinfo.harbor_selfsigned_cert",
          "Value": "True"
        },
        {
          "Key": "guestinfo.harbor_ca",
          "Value": ""
        },
        {
          "Key": "guestinfo.harbor_server_cert",
          "Value": ""
        },
        {
          "Key": "guestinfo.harbor_server_key",
          "Value": ""
        },
        {
          "Key": "guestinfo.network_ip_address",
          "Value": ""
        },
        {
          "Key": "guestinfo.network_netmask",
          "Value": ""
        },
        {
          "Key": "guestinfo.network_gateway",
          "Value": ""
        },
        {
          "Key": "guestinfo.network_dns_server",
          "Value": ""
        },
        {
          "Key": "guestinfo.network_dns_domain",
          "Value": ""
        }
      ],
      "NetworkMapping": [{
        "Name": "nic0",
        "Network": ""
      }],
      "Annotation": "Harbor ova vSphere image - VMware Photon OS 64-bit and Harbor v2.6.3+vmware.1",
      "MarkAsTemplate": false,
      "PowerOn": false,
      "InjectOvfEnv": false,
      "WaitForIP": false,
      "Name": null
    }
    ```

which you can tune to your liking by filling in the actual values for your environment (see following tips)

??? tip
    A few notes about properties in the `PropertyMapping` array:

    !!! note ""
        __Passwords__
        
        The following passwords have to be at least 8 and at most 128 characters long:

        - `guestinfo.root_password`
        - `guestinfo.harbor_admin_password`

        No other complexity requirements have to be met.

    !!! note ""
        __DiskProvisioning__

        Make sure you do specify the correct value for `DiskProvisioning` (i.e. `thin`), or you may hit an error like

        ```text
        govc: deploy error: Invalid value for DatastoreMappingParams.target_provisioning_type: flat.
        ```

    !!! note ""
        __Harbor hostname__

        Set the `guestinfo.harbor_hostname` property to the fully-qualified domain name you want to use to reach the service.

        If the certificate gets auto-generated, this is the value that is used as Common Name (and SAN, too),
        otherwise, it must match with the certificate you provide.

    !!! note ""
        __Harbor scanner__

        Set the `guestinfo.harbor_scanner_enable` property to either `"True"` or `"False"` (as a string),
        to avoid the following error at power on:

        ```text
        govc: Property 'Enable Harbor Default Scanner' must be configured for the VM to power on.
        ```

    !!! note ""
        __Certificates and keys__
        
        CA and server certificates and key must be supplied in plain PEM format, not base64-encoded.
        If no values are provided they will be auto-generated, thus they will be self-signed (not recommended for production use).

    !!! note ""
        __PowerOn__

        If set to `true`, the VM will be powered on as soon as it's created.

    !!! note ""
        __WaitForIP__

        If set to `true`, `govc` will not return until the VM acquires an IP address.

Finally, you run the following command to deploy a new virtual machine (named `registry`):

```sh
govc import.ova -options harbor.json -name registry $HOME/vcc-downloads/${HARBOR_OVA}
```

## Trust your registry

You need to make sure that the Docker client on the bastion host trusts the internal registry.
In fact, when you will get to deploy Tanzu Kubernetes Grid, the Tanzu CLI will use the Docker client to run a local kind cluster,
that will eventually deploy all the required components to the actual location (i.e. your vSphere cluster).
The Docker client will attempt to pull the kind image from the internal registry, and will ultimately fail if the trust is not established.

Docker verifies a registry certificate against system CAs and additional CA certificates at `/etc/docker/certs.d/<REGISTRY-FQDN>/ca.crt`.
If the URL of the registry you want to interact with has a port you need to specify it as well.
See [Docker docs](https://docs.docker.com/engine/security/certificates/) for more information.

So you need to copy the CA certificate into the above location as

```sh
sudo mkdir /etc/docker/certs.d/<REGISTRY-FQDN>
sudo cp /path/to/harbor-ca.pem /etc/docker/certs.d/<REGISTRY-FQDN>/ca.crt
sudo systemctl reload docker
```

## Configure `tkg` project

Tanzu Kubernetes Grid images must be loaded into a dedicated project in Harbor, a hierarchical object that acts as a folder
and allows to set specific configurations in terms of, e.g., permissions and quota.

Navigating to the Harbor UI[^1] as `admin` user, you can follow the `Projects` link, and create a new project hitting the
`+ NEW PROJECT` button.
The pop-up modal window allows you to set the name (we can name it `tkg`), the quota and whether or not it should be publicly accessible (no authentication required).
More granular settings can be applied by editing the project after it has been created.

!!! warning
    For TKG to successfully pull platform images, the `tkg` Harbor project **must be public**.

    If concerned about security implications, please note that

    - images are publicly available anyways at `projects.registry.vmware.com/tkg`;
    - public access means that **unauthorised users** (which includes the **unauthenticated** ones) are allowed to pull but not to push images, thus not allowed to inject malicious code.

## Configure users

You now need to create a user with write permissions to the `tkg` project created [previously](#configure-tkg-project).
You can do so by navigating to the `Administration > Users` menu in the Harbor UI[^1].

Once the user has been created, it needs to be authorised to the `tkg` project, granting it _at least_ the `developer` role:
go back to `Projects`, click on the `tkg` project to edit it and add a user selecting the correct role.
More info on roles at the [official Harbor docs](https://goharbor.io/docs/2.6.0/administration/managing-users/).

[^1]: in the future, I might want to add a section to this guide for configuring Harbor programmatically,
either by `curl`ing APIs, or leveraging tools like Terraform or Ansible.
