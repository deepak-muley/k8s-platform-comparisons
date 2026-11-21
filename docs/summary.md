# Executive summary

What this repo found, as of November 2026. Commands and the object each one writes are in [TLDR](tldr.md). The cell-by-cell matrix is [Comparison](comparison.md).

Seven platforms: OpenShift 4.18/4.19, NKP 2.17–2.19, GKE (and Google Distributed Cloud air-gapped), EKS, EKS Anywhere, AKS with Fleet Manager, and RKE2/Rancher. The same component is a pod roll on one product and a node replacement on another. Look the component up before you promise a same-day patch.

## Can you patch it alone?

Speed 2 is a DaemonSet or Deployment. Speed 3 is a node drain. "Yes" below is the supported split. Details and the exceptions are in [CVE scope](cve-scope.md).

| | OS, same Kubernetes | Control plane, workers stay | Vendor CNI or CSI alone |
| --- | --- | --- | --- |
| OpenShift | No. The z-stream is both. | Pause the worker pool. Unpausing still moves OS and kubelet together. | CNI no. An optional CSI operator can move from a new catalog index. |
| NKP on Nutanix | Only if a second image exists for that Kubernetes version. Still a new VM. | No. The new control-plane machine is the new image. | `nkp upgrade addons`, and only to the build in that NKP release. |
| GKE | Yes. A newer `-gke.N` inside the minor. The VM is still new. | Yes. `--master`. | No. They ride the GKE version. |
| EKS | Yes. New AMI, same Kubernetes version. | Yes. `update-cluster-version`. | Yes. `update-addon`. It does not follow the control plane. |
| EKS Anywhere | New template at the same `kubernetesVersion`. | The control-plane machines roll onto the new image. | CNI no. CSI is yours: `helm upgrade`. |
| AKS | Yes. `--node-image-only`. | Yes. `--control-plane-only`. | No. They ride the AKS version. |
| RKE2 | Yes. The OS package manager. | Yes. Same host. Servers, then agents. | Bundled CNI no. A chart you installed, yes. |

Third-party CSI (Portworx, Trident, and the rest) is `helm upgrade` on every platform here. A new StorageClass is a config write everywhere. Pod CIDR, service CIDR, and replacing the CNI are a new cluster. AKS can migrate to Azure CNI powered by Cilium in place, and that reimages every pool.

## Many clusters

Pick the layer first. A Kubernetes minor, a node-image CVE, and a Prometheus bump are three changes. One window makes the failure unreadable.

| Product | What fans out | What you still build |
| --- | --- | --- |
| AKS Fleet Manager | Staged runs. `Full` or `NodeImageOnly`. A failed stage can stop the rest. | The stage JSON. CSI and Prometheus are not a run. |
| GKE | Release channels. Rapid soaks before Regular. | The choice to join the channel. Google then picks the patch inside the window. |
| OpenShift + ACM | A policy sets `ClusterVersion` on the clusters you label. | The canary pool inside each cluster. |
| NKP | One cluster for Kubernetes. A workspace upgrade moves platform apps on every cluster in that workspace. | The split between prod and nonprod workspaces. |
| Rancher | Cluster upgrades, plus Fleet for manifests and charts. | A target that is only the canary. |
| EKS, EKS Anywhere | One API call or one cluster object. | The loop, the canary, and the stop. |

Inside one cluster the throttle is surge and PodDisruptionBudgets. A budget of `minAvailable: 100%` on a single replica stalls the drain. Find it on the canary.

## Apps

A Helm chart the platform does not own stays where you left it when Kubernetes moves.

NKP is the exception for its default set: Prometheus, cert-manager, Traefik, Flux, and the rest bump together, before Kubernetes. Catalog apps are not in that set. OpenShift monitoring is in the payload and moves with `oc adm upgrade`. Logging is a subscription you approve. GKE and AKS collectors move when the vendor rolls them, often with the node. EKS add-ons, EKS Anywhere packages, and Rancher charts move only when you bump them.

On a hub, Flux, Dex, and Thanos exist twice. After the hub upgrades, the hub copy is newer and the spoke copy is still the old one. The spoke copy is the one pulling, logging in, and scraping. The ranges are in [Compatibility](compatibility.md).

## Air gap

Stage the exact release in the site registry before you change the desired version. Public GKE, EKS, and AKS are not air-gapped products. The disconnected counterparts are Google Distributed Cloud air-gapped, EKS Anywhere, and AKS on Azure Local. OpenShift, NKP, EKS Anywhere, and RKE2 do air-gap on the product you already run. An OpenShift minor mirror is often 30 GB or more. An NKP 2.17 bundle is about 15 GB, and the VM image is a separate import.

## What is still receiving patches

Upstream Kubernetes patches the latest three minors. In November 2026 those are 1.37, 1.36, and 1.35. 1.34 received its final patch on 27 Oct 2026. A kubelet can still be three minors behind the apiserver and already be out of maintenance. OpenShift EUS, EKS extended support, AKS LTS, and a Rancher matrix can keep a minor after that date. The vendor date is the one for that product. The table is in [Compatibility](compatibility.md).

## Where the long version is

| Question | Page |
| --- | --- |
| The command and the object it writes | [TLDR](tldr.md) |
| Core vs add-on, and the four speeds | [Concepts](concepts.md) |
| Ownership, upgrade unit, blast radius | [Comparison](comparison.md) |
| Skew, two copies on the hub, patch end dates | [Compatibility](compatibility.md) |
| Why an NKP node upgrade replaces the VM | [Cluster API](capi.md) |
| What a CVE actually moves | [CVE scope](cve-scope.md) |
| Logging, monitoring, catalog apps | [Platform apps](platform-apps.md) |
| Disconnected install and upgrade | [Air gap](airgap.md) |
| Canary, concurrency, what to record | [Upgrade at scale](scale.md) |
| A design where every row above is its own wave | [Ideal platform, summary](ideal-platform-summary.md) |
