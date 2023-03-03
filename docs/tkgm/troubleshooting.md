# Troubleshooting

<!-- ## SSH into the nodes

## Nodes not joining the cluster -->

## Change Kube-VIP IP address

You should never need to change the IP address of the Kube-VIP load balancer, because it is something you should carefully plan and decide upfront; but in case you do need it, this section may be helpful.

!!! warning
    This is a quite impactful operation, as it may (and will) break the connections between all the nodes.
    In the end you might also need to roll all the worker nodes out, for the change to be fully propagated.
    So, depending on your cluster size, get yourself a maintenance window long enough to rebuild all the nodes.

Do remember that you have to pick an address belonging to the same control-plane nodes network, and make sure it is outside the DHCP range.

Kube-VIP is managed via a static pod running on the control-plane nodes, this means that its definition lies on a plain YAML file on the node's file system.
For kubeadm-based clusters, the static pods manifest files are stored in the `/etc/kubernetes/manifests` directory, and TKG is no exception.

[Log into the control-plane nodes](#ssh-into-the-nodes) and go to the manifests directory.
Print the definition of the Kube-VIP pod, if you wish:

```sh
cd /etc/kubernetes/manifests
cat kube-vip.yaml
```

Now you need to edit the YAML file and set the value of the environment variable named `address` to the new IP address, like:

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: kube-vip
  namespace: kube-system
spec:
  containers:
  - args:
    - manager
    env:
...
    - name: address
      value: 172.31.8.12
...
```

Do the same thing on all the control-plane nodes of the cluster,
and then delete all the `kube-vip-*` pods from the `kube-system` namespace, to get them recreated right away with the new configuration:

```sh
kubectl -n kube-system get pods -o name | grep kube-vip- | xargs kubectl -n kube-system delete
```

You also need to change the ConfigMap `kubeadm-config` in the `kube-system` namespace, to make sure that future nodes will get configured correctly.
Set the key `controlPlaneEndpoint` in `data.ClusterConfiguration` to the new IP address, followed by `:6443` (the standard kube-api port does not change).

## Add names or IPs to API server certificate

If you want to reach out to your Kubernetes cluster using different DNS names and/or IP addresses (i.e. you need to [change the Kube-VIP IP address](#change-kube-vip-ip-address)),
you may also want to add those endpoints as SANs in the API server certificate.

*[SANs]: Subject Alternative Names

First of all, you need to [log into the control-plane nodes via SSH](#ssh-into-the-nodes) and `cd` to the `/etc/kubernetes/pki` directory.
Here you can find all the certificates and keys used by Kubernetes, and the one you care about is `apiserver.crt`.
You can actually take a look at it and make sure it does not include the new endpoints, yet:

```sh
sudo openssl x509 -text < apiserver.crt
```

The `kubeadm` tool is used for the whole lifecycle of the cluster, and it comes in handy in this case, too.
For example, you can check certificates expiration executing

```sh
kubeadm certs check-expiration
```

which shows you also which CA signed the certificates and whether it is external or not.

!!! warning
    The following procedure applies when no external CA has been used to sign the API server certificate.

Kubeadm stores its configuration in a ConfigMap in the cluster, and that's where you start from to add the new endpoints.
Fetch the ConfigMap contents and store it to a file:

```sh
sudo kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' --kubeconfig /etc/kubernetes/admin.conf > /tmp/kubeadm.conf
```

Then, edit the `/tmp/kubeadm.conf` file to add (or edit if already present) the `apiServer.certSANs` key, as follows:

```yaml
apiServer:
  certSANs:
    - 172.31.8.12
    - my.cluster.endpoint.fqdn
```

Make sure you do remove the old certificate and key, otherwise `kubeadm` will not create a new one.
Store them somewhere else rather than deleting them, just in case:

```sh
sudo mv /etc/kubernetes/pki/apiserver.{crt,key} ~
```

Now run the command to build a new certificate and key pair:

```sh
sudo kubeadm init phase certs apiserver --config /tmp/kubeadm.conf
```

The API server picks up the new certificate immediately and you can assert it includes the new SANs.

!!! warning
    If the control-plane is composed of multiple nodes, do make sure you repeat the same process on all of them.
    **DO NOT** copy the certificate across all the nodes, as its SANs include also the IP address of the node itself, which is unique for each node.
    However, you can feed the same `kubeadm.conf` file to the previous `kubeadm` command on all nodes.

Finally, upload the modified kubeadm configuration back to the ConfigMap in the cluster

```sh
kubeadm init phase upload-config kubeadm --config /tmp/kubeadm.conf --kubeconfig /etc/kubernetes/admin.conf
```

## Change cluster configuration

The `Cluster` objects represent the actual Tanzu clusters known to the management cluster, both the management itself and the workload clusters.
A change to those object forces all the nodes of the cluster (control-plane and worker nodes) to be rolled out with the new configuration.

For example, if you need to apply or change the proxy settings to the cluster, you can add/change an item, similar to the following, to the `spec.topology.variables` list:

```yaml
- name: proxy
  value:
    httpProxy: http://proxy.my.domain:proxy-port
    httpsProxy: http://proxy.my.domain:proxy-port
    noProxy:
    - .my.domain
    - 10.0.0.0/8
    - 172.16.0.0/12
    - 192.168.0.0/16
    - localhost
    - 127.0.0.1
    - .svc
    - .svc.cluster.local
    - 100.96.0.0/11
    - 100.64.0.0/13
```

You can then observe the nodes as they get replaced.

!!! warning
    Be aware that this will make the connection drop, as the Kube-VIP will need to be re-negotiated between the old and new node(s),
    so make sure you run this during a proper maintenance window.

<!-- ## Provide custom CA certificates to Pinniped

If your OIDC IdP endpoint exposes a TLS certificate signed by an unknown authority (i.e. own company CA),
you need to provide the CA certificate to pinniped, otherwise the connection will fail.
Same thing if pinniped gets to the IdP endpoint via a TLS-enabled proxy that uses a custom CA -->

## The registry throttles the upload-bundle command

The `tanzu isolated-cluster upload-bundle` command pushes multiple images in parallel to the registry,
and this may cause congestion resulting in upload failures.

As of now, the above command has no flags or modifiers (that I'm aware of) to decrease the number of parallel uploads,
but you can manually push the images leveraging `imgpkg` directly.

The `tanzu isolated-cluster download-bundle` command, used to pull the images, creates the `publish-images-fromtar.yaml` file
to map every tar file to its correspondent OCI image path, and it looks like

```yaml
ako-operator-v1.7.0_vmware.3.tar: ako-operator
ako-v1.8.2_vmware.1.tar: ako
antrea-advanced-debian-v1.7.2_vmware.1.tar: antrea-advanced-debian
azure-cloud-controller-manager-v1.1.26_vmware.1.tar: azure-cloud-controller-manager
azure-cloud-controller-manager-v1.23.23_vmware.1.tar: azure-cloud-controller-manager
azure-cloud-controller-manager-v1.24.10_vmware.1.tar: azure-cloud-controller-manager
azure-cloud-node-manager-v1.1.26_vmware.1.tar: azure-cloud-node-manager
azure-cloud-node-manager-v1.23.23_vmware.1.tar: azure-cloud-node-manager
azure-cloud-node-manager-v1.24.10_vmware.1.tar: azure-cloud-node-manager
calico-all-cni-v3.24.1_vmware.1.tar: calico-all/cni
...
```

So the following snippet can be used to parse this file and push the images sequentially:

```sh
cd /path/to/tkgm-bundle
yq e 'to_entries|map("imgpkg copy --tar " + .key + " --registry-ca-cert-path /path/to/harbor-ca.pem --to-repo ${TKG_CUSTOM_IMAGE_REPOSITORY}/" + .value)|.[]' publish-images-fromtar.yaml | bash
```

This script uses the same info as the [images relocation section](./relocate-images.md), like the `tkgm-bundle` directory and the `harbor-ca.pem` file.
