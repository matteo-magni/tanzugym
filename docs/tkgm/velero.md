# Velero

```sh
vcc download -p vmware_tanzu_kubernetes_grid -s tkg -v 2.1.1 -f velero-linux-v1.9.5+vmware.1.gz --accepteula
gunzip -c ~/vcc-downloads/velero-linux-v1.9.5+vmware.1.gz | sudo tee /usr/local/bin/velero >/dev/null && sudo chmod +x /usr/local/bin/velero
```

Get the available tags for the `aws` plugin:

```sh
imgpkg tag list -i registry.h2o-2-16135.h2o.vmware.com/tkg/velero/velero-plugin-for-aws
```

```sh
velero install --provider aws --plugins "registry.h2o-2-16135.h2o.vmware.com/tkg/velero/velero-plugin-for-aws:v1.5.3_vmware.1" --bucket velero --secret-file ./credentials-velero --backup-location-config "region=minio,s3ForcePathStyle=true,s3Url=https://minio-api.services.h2o-2-16135.h2o.vmware.com" --snapshot-location-config region="default"
```

Get the CSI credentials to a file:

```sh
kubectl -n kube-system get secret vsphere-config-secret -o jsonpath='{.data.csi-vsphere\.conf}'| base64 -d > csi-vsphere.conf
```

and create a secret in the `velero` namespace out of it:

```sh
kubectl -n velero create secret generic velero-vsphere-config-secret --from-file=csi-vsphere.conf
```

Create a `ConfigMap` for Velero plugins:

```sh
kubectl apply -n velero -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: velero-vsphere-plugin-config
data:
  cluster_flavor: TKG
  vsphere_secret_name: velero-vsphere-config-secret
  vsphere_secret_namespace: velero
EOF
```
