# Bastion host

You will need a Linux bastion host equipped with a few tools to be able to fully deploy and operate TKGM.
You can pick any distribution of choice, as long as you know how to do a basic configuration and deploy software on it.
However, in this guide, I provide guidance on Ubuntu and RHEL-like (i.e. RHEL, CentOS, Oracle Linux) distributions only.
It is also recommended not to use the `root` user, but instead create an unprivileged user (belonging to `wheel` group)
and use it for day1 and day2 operations.
In this guide I am using the `tanzu` user, that can be created as

```sh
useradd -m -G wheel tanzu
```

To allow the bastion host to [relocate TKGM images](./relocate-images.md), the minimum requirements for the bastion host are
2GB RAM, 2 vCPU and 45 GB **free** hard disk space, therefore it's better to provide **at least** ~10GB more for OS and packages installation.

If you have fine-grained file systems, make sure you grant at least 20 GB free disk space for the `/tmp` directory,
for storing temporary files during images relocation.

## Proxy configuration (optional)

NSX-T can be used to create segregated networks with proper policies to block egress traffic,
and simulate an air-gapped environment for the TKGM cluster

The bastion host is used to fetch the required images and copy them to the TKGM isolated network,
therefore it somehow needs to access the Internet, i.e. via a proxy.

For simulating a proxy we can use another host running a basic [Squid](http://www.squid-cache.org/) installation.
On Ubuntu, it can be installed as

```sh
apt install -y squid
```

By default, Squid denies all outgoing connections if not explicitly authorised via a `http_access` directive.
The `/etc/squid/conf.d/` directory can be used to store custom configuration files, rather than changing the default `/etc/squid/squid.conf`.

```text title="/etc/squid/conf.d/allowed.conf"
acl external_allowed dstdom_regex "/etc/squid/conf.d/allowed-domains-regex.txt"
http_access allow external_allowed
```

For testing purposes, all domains can be allowed as

```text title="/etc/squid/conf.d/allowed-domains-regex.txt"
.*
```

and with `curl` you can make sure that squid is working

```text hl_lines="7-9"
❯ curl -x http://my-squid-host:3128 -I github.com

HTTP/1.1 301 Moved Permanently
Content-Length: 0
Location: https://github.com/
Date: Thu, 07 Sep 2023 07:49:44 GMT
X-Cache: MISS from my-squid-host
X-Cache-Lookup: MISS from my-squid-host:3128
Via: 1.1 my-squid-host (squid/5.7)
Connection: keep-alive
```

## Software installation

First of all, you need to identify a directory where you want to store those binaries.
In the following guide, we'll be using the `LOCALBIN` variable you can set to whatever directory you are comfortable with,
but make sure you also add it to your user's `PATH` environment variable.
Also, if writing to it requires superuser privileges, do not forget to prepend every write command with `sudo`.

```sh
LOCALBIN="$HOME/.local/bin"
mkdir -p $LOCALBIN
export PATH=$PATH:$LOCALBIN
```

### OS tools

!!! info ""
    === "Oracle Linux"

        ```sh
        sudo dnf install -y bash-completion bind-utils net-tools traceroute
        ```

### `git`

Most distributions come with `git` pre-installed, but in case yours doesn't you can leverage your package manager to get it:

!!! info ""
    === "Oracle Linux"

        ```sh
        sudo dnf install -y git-all
        ```

    === "Ubuntu"

        ```sh
        sudo apt install -y git
        ```

### `kubectl`

The latest `kubectl` release can be installed as

```sh
curl -sSfLO "https://dl.k8s.io/release/$(curl -sSfL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
install kubectl ${LOCALBIN}/kubectl
```

### `kubectx` and `kubens`

Helper scripts to easily switch kubeconfig context and set default namespace.
You may want to get the scripts from an actual release and not from the `main` branch.

```sh
LATEST="v0.9.4"
sudo git clone --depth 1 --branch $LATEST -c advice.detachedHead=false https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -sf /opt/kubectx/kubectx ${LOCALBIN}/kubectx
sudo ln -sf /opt/kubectx/kubens ${LOCALBIN}/kubens
```

### `kubectl-tree`

Show resources hierarchy

```sh
curl -sSfL https://github.com/ahmetb/kubectl-tree/releases/download/v0.4.3/kubectl-tree_v0.4.3_linux_amd64.tar.gz | tar xzf - kubectl-tree
sudo install kubectl-tree ${LOCALBIN}
```

### helm

```sh
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | HELM_INSTALL_DIR=~/.local/bin bash
```

### `fzf`

[`fzf`](https://github.com/junegunn/fzf) is a general-purpose command-line fuzzy finder.

```sh
curl -sSfL https://github.com/junegunn/fzf/releases/download/0.38.0/fzf-0.38.0-linux_amd64.tar.gz | tar xzf -
install fzf ${LOCALBIN}/fzf
```

### `k9s`

```sh
curl -sSfL https://github.com/derailed/k9s/releases/download/v0.27.3/k9s_Linux_amd64.tar.gz | tar xzf - k9s
install k9s ${LOCALBIN}/k9s
```

### `stern`

Stern is a CLI tool to tail logs from multiple pods and multiple containers at the same time.

```sh
curl -sSfL https://github.com/stern/stern/releases/download/v1.24.0/stern_1.24.0_linux_amd64.tar.gz | tar xzf - stern
sudo install stern ${LOCALBIN}
```

### `yq`

```sh
curl -sSfL -o yq https://github.com/mikefarah/yq/releases/download/v4.30.8/yq_linux_amd64
install yq ${LOCALBIN}/yq
```

### `jq`

!!! info ""
    === "Oracle Linux"

        ```sh
        sudo dnf install -y jq
        ```

    === "Ubuntu"

        ```sh
        sudo apt install -y jq
        ```

### docker

!!! info ""
    === "Oracle Linux"

        ```sh
        sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
        sudo dnf update -y
        sudo dnf install -y docker-ce docker-ce-cli containerd.io
        ```

Start and enable the `docker` service:

```sh
sudo systemctl start docker
sudo systemctl enable docker
```

Finally, add the user to the `docker` group:

```sh
sudo gpasswd -a tanzu docker
```

### `vcc`

`vcc` is a useful CLI tool for downloading packages from customerconnect.vmware.com.
Get the latest release from <https://github.com/vmware-labs/vmware-customer-connect-cli/releases>, i.e.:

```sh
curl -sSfL -o vcc https://github.com/vmware-labs/vmware-customer-connect-cli/releases/download/v1.1.2/vcc-linux-v1.1.2
install vcc ${LOCALBIN}/vcc
```

Some of the following steps require you to download software from Customer Connect.
It is therefore necessary to set authentication credentials for `vcc` as environment variables `VCC_USER` and `VCC_PASS`, i.e.:

```sh
export VCC_USER="my-email-address@my-domain.com"
export VCC_PASS='secret-P4ss'
```

Just make sure that

- you surround values with single-quotes and escape single-quotes characters in them.
- multi-factor authentication is **disabled** in your customerconnect profile, otherwise the CLI tool won't work (yet)

### `govc`

`govc` allows you to interact with the vSphere vCenter appliance via CLI, thus being able to script your way to the configuration of your environment.
Select the proper version/platform from <https://github.com/vmware/govmomi/releases> and install it, i.e.:

```sh
curl -sSfL https://github.com/vmware/govmomi/releases/download/v0.30.2/govc_Linux_x86_64.tar.gz | tar xzf - govc
install govc ${LOCALBIN}/govc
```

### Carvel suite

The Carvel suite is a set of useful tools for dealing with YAML files and OCI images.

```sh
curl -L https://carvel.dev/install.sh | K14SIO_INSTALL_BIN_DIR=${LOCALBIN} bash
```

### `tanzu` CLI

The `tanzu` CLI is available within the TKG product.
Using `vcc` you can browse the products/subproducts/versions/files hierarchy and download exactly what you need.
Make sure you choose the correct file for your platform (i.e. `linux-amd64`).

For example,

```sh
# list all the available products
vcc get products

# list all the available subproducts belonging to the vmware_tanzu_kubernetes_grid product
vcc get subproducts -p vmware_tanzu_kubernetes_grid

# list all the versions for the subproduct tkg
vcc get versions -p vmware_tanzu_kubernetes_grid -s tkg

# list all the files available for version 2.1.0
# at this point vcc does the actual login
vcc get files -p vmware_tanzu_kubernetes_grid -s tkg -v 2.1.0

# download the tanzu cli
vcc download -p vmware_tanzu_kubernetes_grid -s tkg -v 2.1.0 -f 'tanzu-cli-bundle-linux-amd64.tar.gz' --accepteula
```

The downloaded files are available in the `~/vcc-downloads` directory.

Now you can install the `tanzu` CLI:

```sh
tar xvzf vcc-downloads/tanzu-cli-bundle-linux-amd64.tar.gz
install cli/core/v0.28.0/tanzu-core-linux_amd64 ${LOCALBIN}/tanzu
```

and the bundled plugins:

```sh
tar xzf tanzu-framework-plugins-standalone-linux-amd64.tar.gz -C cli
tanzu plugin install all --local cli/standalone-plugins

tar xzf tanzu-framework-plugins-context-linux-amd64.tar.gz -C cli
tanzu plugin install all --local cli/context-plugins
```

Now you can get rid of the `cli` directory:

```sh
rm -rf cli
```
