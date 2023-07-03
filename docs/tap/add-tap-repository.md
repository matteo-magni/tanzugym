# Add TAP kapp repository

You can now make your relocate images available to your cluster.
Some environment variables used in this guide have already been defined in the [relocate TAP images](./relocate-tap-images.md) section,
so do make sure you follow along.

First of all, create a namespace where you plan to deploy TAP:

```sh
kubectl create ns tap-install || true
```

Then create a secret that will be used to authenticate to the registry and pull TAP images:

```sh
tanzu secret registry add tap-registry \
    --server   ${IMGPKG_REGISTRY_HOSTNAME_1} \
    --username ${IMGPKG_REGISTRY_USERNAME_1} \
    --password ${IMGPKG_REGISTRY_PASSWORD_1} \
    --namespace tap-install \
    --export-to-all-namespaces \
    --yes
```

!!! important
    Use a service account with read permissions to the `${TAP_REPOSITORY}` repository

Add a Tanzu repository to make TAP packages available:

```sh
tanzu package repository add tanzu-tap-repository \
  --url ${IMGPKG_REGISTRY_HOSTNAME_1}/${TAP_REPOSITORY}:${TAP_VERSION} \
  --namespace tap-install
```

Verify that the repository has been reconciled successfully

```sh
❯ tanzu package repository get tanzu-tap-repository --namespace tap-install

NAMESPACE:               tap-install
NAME:                    tanzu-tap-repository
SOURCE:                  (imgpkg) registry.my.domain/tap/tap-packages:1.5.2
STATUS:                  Reconcile succeeded
CONDITIONS:              - type: ReconcileSucceeded
  status: "True"
  reason: ""
  message: ""
USEFUL-ERROR-MESSAGE:
```

and the packages are available

```sh
❯ tanzu package available list --namespace tap-install

  NAME                                                 DISPLAY-NAME
  accelerator.apps.tanzu.vmware.com                    Application Accelerator for VMware Tanzu
  api-portal.tanzu.vmware.com                          API portal
  apis.apps.tanzu.vmware.com                           API Auto Registration for VMware Tanzu
  apiserver.appliveview.tanzu.vmware.com               Application Live View ApiServer for VMware Tanzu
  app-scanning.apps.tanzu.vmware.com                   Supply Chain Security Tools - App Scanning (Alpha)
  application-configuration-service.tanzu.vmware.com   Application Configuration Service
  backend.appliveview.tanzu.vmware.com                 Application Live View for VMware Tanzu
  bitnami.services.tanzu.vmware.com                    bitnami-services
  buildservice.tanzu.vmware.com                        Tanzu Build Service
  carbonblack.scanning.apps.tanzu.vmware.com           VMware Carbon Black for Supply Chain Security Tools - Scan
  cartographer.tanzu.vmware.com                        Cartographer
  cnrs.tanzu.vmware.com                                Cloud Native Runtimes
  connector.appliveview.tanzu.vmware.com               Application Live View Connector for VMware Tanzu
  controller.source.apps.tanzu.vmware.com              Tanzu Source Controller
  conventions.appliveview.tanzu.vmware.com             Application Live View Conventions for VMware Tanzu
  crossplane.tanzu.vmware.com                          crossplane
  developer-conventions.tanzu.vmware.com               Tanzu App Platform Developer Conventions
  eventing.tanzu.vmware.com                            Eventing
  external-secrets.apps.tanzu.vmware.com               External Secrets Operator
  fluxcd.source.controller.tanzu.vmware.com            Flux Source Controller
  grype.scanning.apps.tanzu.vmware.com                 Grype for Supply Chain Security Tools - Scan
  learningcenter.tanzu.vmware.com                      Learning Center for Tanzu Application Platform
  metadata-store.apps.tanzu.vmware.com                 Supply Chain Security Tools - Store
  namespace-provisioner.apps.tanzu.vmware.com          Namespace Provisioner
  ootb-delivery-basic.tanzu.vmware.com                 Tanzu App Platform Out of The Box Delivery Basic
  ootb-supply-chain-basic.tanzu.vmware.com             Tanzu App Platform Out of The Box Supply Chain Basic
  ootb-supply-chain-testing-scanning.tanzu.vmware.com  Tanzu App Platform Out of The Box Supply Chain with Testing and Scanning
  ootb-supply-chain-testing.tanzu.vmware.com           Tanzu App Platform Out of The Box Supply Chain with Testing
  ootb-templates.tanzu.vmware.com                      Tanzu App Platform Out of The Box Templates
  policy.apps.tanzu.vmware.com                         Supply Chain Security Tools - Policy Controller
  scanning.apps.tanzu.vmware.com                       Supply Chain Security Tools - Scan
  service-bindings.labs.vmware.com                     Service Bindings for Kubernetes
  services-toolkit.tanzu.vmware.com                    Services Toolkit
  snyk.scanning.apps.tanzu.vmware.com                  Snyk for Supply Chain Security Tools - Scan
  spring-boot-conventions.tanzu.vmware.com             Tanzu Spring Boot Conventions Server
  spring-cloud-gateway.tanzu.vmware.com                Spring Cloud Gateway
  sso.apps.tanzu.vmware.com                            AppSSO
  tap-auth.tanzu.vmware.com                            Default roles for Tanzu Application Platform
  tap-gui.tanzu.vmware.com                             Tanzu Application Platform GUI
  tap-telemetry.tanzu.vmware.com                       Telemetry Collector for Tanzu Application Platform
  tap.tanzu.vmware.com                                 Tanzu Application Platform
  tekton.tanzu.vmware.com                              Tekton Pipelines
  workshops.learningcenter.tanzu.vmware.com            Workshop Building Tutorial
```
