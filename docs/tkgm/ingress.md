# Ingress controller

## Kube-VIP Load Balancer

This topic describes using Kube-VIP as a load balancer for workloads hosted on Tanzu Kubernetes Grid (TKG) workload clusters.

!!! warning "Technical Preview"
    This feature is still a technical preview, yet very helpful to quickly create a load balancer to be used in a lab environment,
    and that will allegedly be fully supported on TKG future releases.

To learn how to set it up see [official VMware docs](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.1/tkg-deploy-mc-21/mgmt-reqs-network-kube-vip.html).

## NGINX ingress controller

You can install the NGINX ingress controller in a number of ways, detailed in the [NGINX docs website](https://docs.nginx.com/nginx-ingress-controller/installation/).

### Installation via Helm chart

You can pull the NGINX ingress sources from GitHub

```sh
git clone https://github.com/nginxinc/kubernetes-ingress.git --branch v3.0.2
cd kubernetes-ingress/deployments/helm-chart
```

or install a Helm repo and pull the chart

```sh
helm repo add nginx-stable https://helm.nginx.com/stable
helm repo update
helm pull nginx-stable/nginx-ingress --destination . --untar
cd nginx-ingress
```

Either way, if your TKG instance lives in an air-gapped environment, you have to download the helm chart from an Internet-connected host
and then copy it over to the bastion host inside the environment.

You also need to relocate the `nginx/nginx-ingress` image to the internal registry.
Don't forget to provide DockerHub authentication credentials (create a read-only temporary Personal Access Token) to avoid [getting throttled](https://docs.docker.com/docker-hub/download-rate-limit/).

```sh
export IMGPKG_REGISTRY_HOSTNAME_0='<REGISTRY-FQDN>'
export IMGPKG_REGISTRY_USERNAME_0='<REGISTRY-USERNAME>'
export IMGPKG_REGISTRY_PASSWORD_0='<REGISTRY-PASSWORD>'

export IMGPKG_REGISTRY_HOSTNAME_1='index.docker.io'
export IMGPKG_REGISTRY_USERNAME_1='<DOCKERHUB-USERNAME>'
export IMGPKG_REGISTRY_PASSWORD_1='<DOCKERHUB-PAT>'

imgpkg copy -i nginx/nginx-ingress:3.0.2 --to-repo registry.my.domain/apps/nginx-ingress --registry-ca-cert-path harbor-ca.pem
```

Provide a values file for the helm installation to specify your custom settings, i.e.:

- the URL to the relocated image;
- the `service.externalTrafficPolicy` set to `Cluster`;
- the service type and IP address, to make sure it is static and won't change.

```yaml
controller:
  image:
    repository: registry.my.domain/apps/nginx-ingress
    tag: 3.0.2

  service:
    type: LoadBalancer
    loadBalancerIP: 172.31.9.21
    externalTrafficPolicy: Cluster
```

??? info "Brief explanation about the `externalTrafficPolicy`"
    The helm chart's default value is `Local`, which allows to preserve the client IP address but also keeps the connection local to the node,
    with no chance for it to be routed to other nodes in the cluster.

    However, with the default setting, using a load balancer such as Kube-VIP that allocates IP addresses to the control plane nodes (thus assigned to the services of type `LoadBalancer`),
    makes it impossible for external connections to reach the actual application pods, as they do not and never will run on any control plane nodes.

    See also [Kubernetes docs](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/#preserving-the-client-source-ip).

You can also customise it further if you want; all the available parameters can be listed as

```sh
helm show values .
```

Finally, you can start the installation:

```sh
helm install nginx-ingress . --values /path/to/the/values/file.yaml
```

## Deploy test HTTP application

For testing, you can deploy a NGINX web server and expose its default welcome page through the ingress controller, to check connectivity from both within the cluster and outside of it.

First of all, you have to relocate the image (`nginx/nginx:stable-alpine`, in this example) to the internal registry like you did for the ingress controller image.

Then, you can create a Deployment and its Service

```yaml title="deployment.yaml"
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: app1
  name: app1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - image: registry.my.domain/apps/nginx:stable-alpine
        name: nginx
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: app1
  name: app1
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: app1
```

and an Ingress resource accordingly

```yaml title="ingress.yaml"
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app1
  annotations:
    nginx.org/rewrites: "serviceName=app1 rewrite=/"
spec:
  ingressClassName: nginx
  rules:
  - host: "*.my.domain"
    http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1
            port:
              number: 80
```

In this example, the Ingress resource uses a wildcard to listen on all hosts (that end up being associated to the ingress controller IP address),
and allows you to define multiple paths that can route the traffic to multiple backend applications.

You should now create a DNS record to associate `app1.my.domain` to the ingress controller service's IP address `172.31.9.21`,
or you could just forge the `Host` header in the test HTTP request:

```sh
❯ curl -H "Host: app1.my.domain" http://172.31.9.21/app1

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

This proves that the configuration is correct.
