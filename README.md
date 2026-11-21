# Kubernetes platform install and upgrade

Hands-on comparison of how these platforms are installed, how configuration is changed after day one, and how CSI, CNI, and Kubernetes itself are upgraded. Connected and air-gapped paths are both covered, including upgrades across many clusters.

Documented November 2026. Commands were checked against OpenShift 4.18/4.19, NKP 2.17–2.19, current GKE and Google Distributed Cloud air-gapped, Amazon EKS, EKS Anywhere, AKS with Fleet Manager, and RKE2/Rancher. Minor releases move flags. Dry-run, then apply.

Start here: [Executive summary](docs/summary.md).

## Platforms

| Doc | What it is |
| --- | --- |
| [OpenShift](docs/platforms/openshift.md) | Self-managed OpenShift Container Platform. Short notes where ROSA/ARO change who runs the upgrade. |
| [NKP](docs/platforms/nkp.md) | Nutanix Kubernetes Platform. CAPX and the other Cluster API providers under the CLI, Kommander for the fleet. |
| [GKE](docs/platforms/gke.md) | Google Kubernetes Engine. Google Distributed Cloud air-gapped is the disconnected counterpart. |
| [EKS](docs/platforms/eks.md) | Amazon EKS in AWS. Managed add-ons have their own version and a fast update path. |
| [EKS Anywhere](docs/platforms/eks-anywhere.md) | The on-prem, air-gapped EKS distro. You run the machines. |
| [AKS](docs/platforms/aks.md) | Azure Kubernetes Service, plus Fleet Manager for upgrades at scale. |
| [RKE2 and Rancher](docs/platforms/rke2-rancher.md) | The other common on-prem distro with a real air-gap story. Useful next to NKP and EKS Anywhere. |

## How to read this

1. [Executive summary](docs/summary.md) — the findings on one page.
2. [TLDR](docs/tldr.md) — install and upgrade commands, and the CRD or API object each one writes.
3. [Concepts](docs/concepts.md) — core vs add-on, and the four speeds a change can have.
4. [Comparison](docs/comparison.md) — one matrix for ownership, upgrade unit, and blast radius.
5. [Compatibility](docs/compatibility.md) — how far behind a kubelet, a Cluster API workload, or a cluster-local Flux may sit, how far ahead the management copy of Flux, Dex, and Thanos is allowed to be, and when each Kubernetes minor stops receiving patches. Each cell links the page that states it.
6. [Versioning](docs/versioning.md) — major, minor, patch, OS rebuild, and add-on rebuild, and which digit each vendor increments.
7. [Cluster API](docs/capi.md) — why NKP on CAPX replaces the VM, and which products can move Kubernetes without the OS image.
8. [CVE scope](docs/cve-scope.md) — a patch for one component versus a patch that moves Kubernetes and the node OS together.
9. [Platform apps](docs/platform-apps.md) — logging, monitoring, and catalog apps, and which products upgrade them when the platform version moves.
10. The platform page for the product you operate.
11. [Air gap](docs/airgap.md) and [Upgrade at scale](docs/scale.md) when you are doing either for real.
12. [Ideal platform, summary](docs/ideal-platform-summary.md) — one page. The full design is [Ideal platform](docs/ideal-platform.md): every layer on its own wave, apps from a hub to every spoke, and one grid for thousands of clusters.

## The distinction that matters

Every platform has the same three layers. The product names differ.

| Layer | What it contains | Who moves it |
| --- | --- | --- |
| Kubernetes core | API server, etcd, scheduler, controller-manager, kubelet | The platform upgrade. On GKE, EKS, and AKS you never touch etcd. |
| Platform core | The CNI the vendor ships, the CSI for that infrastructure, node image, ingress, machine config | Usually the platform release. Sometimes a separate command that still uses the vendor's bundled version. |
| Add-on | Anything you subscribed to, Helm-installed, or enabled from a catalog | You. This is the fast path, and it is only fast when the vendor does not own that component. |

CSI and CNI are not automatically "add-ons". On EKS, the VPC CNI and the EBS CSI driver are managed add-ons with their own versions, so you can roll them without replacing nodes. On OpenShift, the cluster network is a payload operator and moves with the release. On EKS Anywhere, Cilium moves with the cluster upgrade, and CSI is yours. Treat each component as core or add-on only after you look it up on that platform. The [comparison](docs/comparison.md) is the lookup.
