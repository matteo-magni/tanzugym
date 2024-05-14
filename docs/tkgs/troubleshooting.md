# Troubleshooting

## Inspect supervisor cluster

To better troubleshoot problems on the platform, you can inspect what's going on on the supervisor cluster.

The kubeconfig file and credentials for the `kubernetes-admin` user (granted the `cluster-admin` ClusterRole) is only stored in the Supervisor cluster's VMs.
You can SSH to either of those VMs as long as

- network connectivity is established
- you have the `root` password.

To get the password, you need to SSH to the vCenter appliance as the `root` user, and run the following command:

```sh
/usr/lib/vmware-wcp/decryptK8Pwd.py
```

This returns an output like

```text
Read key from file

Connected to PSQL

Cluster: domain-c8:d0cd4264-2079-4d97-a7b1-09eaf1c561a6
IP: 10.220.5.215
PWD: TlA}xwa>lZw89e(1
------------------------------------------------------------
```

Which gives you both the password and the endpoint to connect to (N.B. this is on the Supervisor's management network) as the `root` user.

!!! note
    If your default authentication mechanism is via SSH keys, do make sure you force password authentication, as
    ```sh
    ssh -l root -o PasswordAuthentication=yes -o PubkeyAuthentication=no -o PreferredAuthentications=password $SUPERVISOR_VM_IP
    ```

The kubeconfig file is already available to the root user at `~/.kube/config`, so you can run `kubectl` (the `k` alias is also configured OOTB) commands right away, for example

```sh
root@4201cccd09d37b0eb96bbc105d70a0a6 [ ~ ]# k get no
NAME                               STATUS   ROLES                  AGE   VERSION
4201890d94e215dd90689910b4e0882a   Ready    control-plane,master   21m   v1.25.6+vmware.wcp.2
4201cccd09d37b0eb96bbc105d70a0a6   Ready    control-plane,master   35m   v1.25.6+vmware.wcp.2
4201f59df591b3b158cd4170458a1b26   Ready    control-plane,master   21m   v1.25.6+vmware.wcp.2
```
