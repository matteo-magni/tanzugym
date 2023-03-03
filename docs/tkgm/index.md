# Preface

The environment I'm testing TKG on is composed of a vSphere infrastructure (ESXi + vCenter),
deployed along with a NSX-T platform to provide dynamic network segmentation and management.

NSX-T is not strictly mandatory to follow along, it's just convenient for me for setting up the network
segments, routers and policies to my own liking without the need for physical devices,
as well as the ability to configure everything via API,
that makes me able to provision an entire stack automatically within minutes.

This section is about deploying and managing a [Tanzu Kubernetes Grid 2.1 platform with a Standalone Management Cluster](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.1/tkg-deploy-mc-21/mgmt-index.html).

!!! warning "What this guide is NOT meant for"

    - deploying vSphere with Tanzu
    - giving comprehensive training around Kubernetes
    - deep-diving into the tools used (i.e. Linux, bash, govc, Terraform, ...)

However, some snippets are provided as a reference that you will need to adjust to your own environment,
whilst some others are even ready to be used OOTB.

!!! success "Feedbacks are welcome"
    If you think that something is not clear enough or totally wrong,
    please do [create an issue](https://github.com/matteo-magni/tanzugym/issues) and let me know.
