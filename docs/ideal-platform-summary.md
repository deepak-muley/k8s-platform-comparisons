# Ideal platform, summary

A design, not a product. The long version is [Ideal platform](ideal-platform.md).

Patch the OS, Kubernetes, the kubelet, CSI, CNI, the cloud controller manager, and apps such as Prometheus or Flux on separate clocks. Declare an app once on the hub. Every selected spoke pulls it. A thousand clusters still fit on one screen: rings down the side, components across the top, a version in each cell.

## The job

Publish the artifact. Open one wave. Watch the canary gate. A wave names one component. Rollback is the same wave pointed back at the old version.

| Fix | What moves | How long the canary takes |
| --- | --- | --- |
| CSI, CNI, CCM, Prometheus, Flux | That chart's pods | Minutes |
| Kernel or node image | Nodes, drained | A drain per node |
| Kubernetes minor | A queue: apps that need the new APIs, control plane, CCM if its window requires it, kubelet, then kube-proxy | Control plane first. Workers stay until the kubelet wave. |

A DaemonSet fix never reboots a node. A kernel fix never bumps Prometheus. Skew stays inside [Compatibility](compatibility.md): kubelet at most three minors behind the apiserver and never ahead, CCM at most one minor behind and never ahead.

## Image or bundle

The OS image is the slow artifact: a build, a mirror, a drain per node. It holds the kernel, firmware, and the boot userspace, plus a kernel module when a chart needs one. The kubelet and the container runtime are host packages on that image, so a runtime CVE drains the node and leaves the kernel digest where it is. CNI, CSI, CCM, kube-proxy, CoreDNS, and apps are charts. A chart rolls pods. The long version is [Image or bundle](ideal-platform.md#image-or-bundle).

| OS image, speed 3 | Host package, speed 3, image stays | Chart, speed 2, nodes stay |
| --- | --- | --- |
| Kernel, firmware, boot userspace, a kernel module the chart needs | Kubelet, container runtime | CNI, CSI, CCM, kube-proxy, CoreDNS, Prometheus, Flux |

A new node is composed from those pins. A wave moves one of them. The control plane is its own images, and workers stay.

## The screen

| Ring | Clusters | os | kubernetes | kubelet | cni | csi | ccm | prometheus |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| canary | 5 | 2026.11.4 | 1.36.3 | 1.36.3 | cilium 1.17.2 | ebs 1.41.0 | aws-ccm 1.36.1 | 82.13.6 |
| early | 50 | 2026.11.4 | 1.36.3 | 1.36.3 | cilium 1.17.2 | ebs 1.40.0 → 1.41.0 | aws-ccm 1.36.1 | 82.13.6 |
| broad | 900 | 2026.11.4 | 1.36.3 | 1.35.8 | cilium 1.17.2 | ebs 1.40.0 | aws-ccm 1.36.1 | 82.13.6 |

One column is moving (CSI). Broad's kubelet is still one minor behind because that ring was not promoted. Everything else is one version. Open a cluster only when a cell is bad.

## Hub and spokes

Git on the hub is the source of truth. The hub publishes one artifact per profile. Spokes pull and report the version that is actually running. A newer copy on the hub does not patch the spoke. The grid shows the spoke.

## Limits

A node image still drains a node at a time. Pod CIDR and swapping the CNI plugin are still a new cluster. Versions above are an example, not a release to install.
