# Core, add-on, and how fast a change is

Use this page before the platform pages. The same words mean different objects on each product.

## Three layers

### Kubernetes core

The control plane and the kubelet.

- OpenShift, NKP, EKS Anywhere, and RKE2 run this on machines you (or Cluster API) create. An upgrade replaces or restarts those machines.
- GKE, EKS, and AKS run the control plane for you. You upgrade a version field. Google, AWS, or Azure replace the API server and etcd. You still roll the nodes, except on GKE Autopilot and EKS Auto Mode, where the vendor rolls those too.

### Platform core

Components the product will not form a cluster without, shipped and versioned by the vendor.

Typical members:

- The CNI the installer lays down (OVN-Kubernetes, Cilium, Calico, VPC CNI, Azure CNI).
- The CSI driver for the infrastructure the installer targets (Nutanix Volumes on NKP-on-AHV, EBS on EKS, GCE PD on GKE, Azure Disk on AKS).
- The node OS image or MachineConfig that the kubelet boots from.
- The ingress the console or the platform apps sit behind.
- Cluster API controllers, when the product is built on CAPI (NKP, EKS Anywhere).

Platform core has a compatibility matrix. You can sometimes upgrade it with its own command (`nkp upgrade addons`, `aws eks update-addon`, an OLM channel inside the same OpenShift minor). You cannot pick an arbitrary upstream version and expect support.

### Add-on

A component you chose, installed as an operator, a Helm release, an EKS add-on you opted into, a Kommander catalog app, or a Rancher chart.

You own the version. You can move it on a faster cadence than Kubernetes, as long as you stay inside the Kubernetes version skew the chart supports. The vendor will not roll it forward during a cluster upgrade unless you asked for that (Kommander workspace upgrade, an OLM `Automatic` approval, a GKE add-on that Google manages).

Third-party CSI (Portworx, NetApp Trident, Dell, Ceph) is an add-on on every platform in this repo, including the managed ones. That is the one storage path that is fast everywhere: `helm upgrade` of a controller Deployment and a node DaemonSet. No new VM image.

## Four speeds

When someone asks for a "fast" CNI or CSI change, name which speed they actually get.

| Speed | What happens | How long | Example |
| --- | --- | --- | --- |
| 1. Config write | A CR, add-on API, or Helm value changes. Pods are not restarted, or only the controller is. | Seconds to a couple of minutes | EKS `configurationValues` for an env var. OpenShift `Ingress` controller replicas. GKE maintenance window. NKP `AppDeployment` values that only change a ConfigMap the app reloads. |
| 2. Workload rollout | A Deployment or DaemonSet rolls. Nodes stay. Existing pods on a node drop briefly while the DaemonSet pod restarts. | Minutes per cluster, longer if the DaemonSet has a slow readiness probe | `aws eks update-addon` for `vpc-cni` or `aws-ebs-csi-driver`. `helm upgrade` of a third-party CSI. OLM CSV for an optional operator. RKE2 HelmChart manifest in `/var/lib/rancher/rke2/server/manifests`. |
| 3. Node roll | The node OS, kubelet, or a host-level config changes. The platform cordons, drains, and replaces or reboots the node. | Tens of minutes per batch. A large pool is hours unless surge is raised. | OpenShift `MachineConfig`. `nkp upgrade cluster nutanix --vm-image`. GKE node pool surge or blue-green. `az aks nodepool upgrade`. EKS Anywhere machine rollout. `rke2` service restart on each node. |
| 4. Rebuild | The field is fixed at create. The supported change is a new cluster and a workload move. | Days, because it is a migration | Pod and service CIDR. EKS Anywhere `cniConfig` (Cilium vs kindnet). OpenShift `networkType`. The infrastructure provider of an NKP cluster. Most immutable node-pool fields (machine type, OS disk, subnet) — those are a new pool plus a drain, which is a controlled form of rebuild rather than an in-place edit. |

Speed 2 is the fast path people want for CSI and CNI. You only get it when that component is an add-on or a managed add-on with its own version. If it is platform core baked into the node image, the same "upgrade the CNI" request is speed 3.

Speed 3 is not one mechanism. OpenShift rebases RHCOS on the existing VM and reboots it. NKP on CAPX deletes the VM and clones a new one from a Prism image that already contains that Kubernetes version, so the OS image and the kubelet move together. RKE2 installs a new version on the host you already have, and the OS package manager is a separate step. The matrix is [Cluster API](capi.md).

## What is usually immutable

These are create-time choices on every platform here. Confirm in the platform page before you promise a customer they can change them later.

- Pod CIDR and service CIDR.
- The identity of the CNI plugin, once the cluster is up. AKS is the exception that has a supported in-place data-plane migration to Azure CNI powered by Cilium, and that migration reimages every node pool.
- Cloud provider / infrastructure provider.
- etcd encryption provider, on the distros that offer it, once secrets have been written. OpenShift can re-encrypt, but it is a project, not a flag flip.
- Cluster name and the API endpoint DNS name you gave installers and clients.

## Where configuration lives

| Platform | Source of truth for the cluster | Source of truth for an add-on |
| --- | --- | --- |
| OpenShift | `install-config.yaml` at install, then the cluster itself: `ClusterVersion`, `Network`, `Ingress`, `MachineConfig`, cloud CRs | `Subscription` + `OperatorGroup` for OLM. Helm for anything outside OLM. |
| NKP | `nkp create cluster` flags, or the CAPI / `NKPCluster` objects they create. ClusterClass on current releases. | Kommander `AppDeployment` and Helm releases. Core add-ons are `ClusterResourceSet` objects. Edits to those are overwritten by `nkp upgrade addons`. |
| GKE | GKE API. Terraform, Config Connector, or `gcloud` are clients of that API. | Add-on flags on the cluster resource. Third-party charts live in the cluster and GKE will not reconcile them. |
| EKS | EKS API (`cluster`, `nodegroup`, `addon`). | `addon` resource: `addonVersion` + `configurationValues`. In-cluster edits survive an update only with `resolve-conflicts PRESERVE`. |
| EKS Anywhere | The `Cluster` YAML (`anywhere.eks.amazonaws.com/v1alpha1`) on the admin machine, applied to the management cluster. | Curated packages, or Helm you installed. The cluster spec does not reconcile your Helm releases. |
| AKS | AKS API. | AKS add-on profiles and Azure CNI mode on the managed cluster resource. Third-party charts are yours. |
| RKE2 | `/etc/rancher/rke2/config.yaml` on each server, plus manifests dropped into the server manifests directory. | HelmCharts in that manifests directory, or charts Rancher Fleet applies. |

If the source of truth is a controller that reconciles, a hand edit in the cluster will be reverted or dropped on the next upgrade. That is the NKP `ClusterResourceSet` case and the EKS add-on case without `PRESERVE`. Put the change in the source of truth.

## Upgrade order, the pattern

The products use different commands. The safe order is the same.

1. Mirror or stage the artifacts first when the cluster cannot pull from the internet. The cluster should not discover the next version on its own.
2. Upgrade the fleet/management plane before the clusters it manages (Kommander, ACM, Rancher, the EKS Anywhere management cluster, the GKE fleet host if you use one).
3. Upgrade platform apps and optional operators so they understand the next Kubernetes APIs.
4. Upgrade the control plane.
5. Upgrade nodes, one pool or one surge batch at a time.
6. Upgrade platform-core CNI/CSI if that product does not do it inside step 4 or 5. On EKS this is a separate add-on update and it does not happen automatically when the control plane moves.

Skip a step and the failure mode is specific: Kommander apps that call removed APIs, an OLM operator whose CSV is missing from the next catalog, an EKS add-on left on a version the new Kubernetes minor rejects, a node image that does not contain the kubelet the control plane expects.

## Version skew you can actually run

| Platform | Control plane vs node | Add-on vs cluster |
| --- | --- | --- |
| Kubernetes itself | Node may be up to 3 minor versions older than the control plane on modern Kubernetes (the skew policy widened; confirm the minor you run). Vendors are often stricter than upstream. | — |
| OpenShift | Nodes trail the payload by the time a `MachineConfigPool` takes to roll. Do not pause a pool across a minor upgrade and leave it there. | Optional operators must have a CSV valid for the target minor before you move `ClusterVersion`. |
| NKP | One NKP minor at a time. The management cluster moves first. | Platform apps are upgraded with `nkp upgrade workspace` or `nkp upgrade platform-apps` before the Kubernetes version. Core add-ons are bundled; you do not run a newer CNI than the NKP release you are on. |
| GKE | Nodes can lag the control plane inside the channel's skew window. Auto-upgrade exists so they do not lag past it. | Google-managed CSI/CNI track the GKE version. Your own charts must match the node minor. |
| EKS | Node groups and Fargate can lag; AWS enforces the skew when you call `update-cluster-version`. | Add-ons do not move when the cluster minor moves. You pick a compatible `addonVersion` yourself. |
| EKS Anywhere | No minor skips. Management components may be at most one EKS Anywhere minor ahead of the cluster components. | Cilium is inside the cluster version. Your CSI chart is independent and can move any day. |
| AKS | Node pools can be a minor behind the control plane inside the supported skew. | Azure CSI and Azure CNI move with the AKS version more than as a free-floating chart. Node image can move alone (`--node-image-only`). |
| RKE2 | Upgrade one minor at a time. Servers before agents. All servers should land on the new version before agents do. | Charts you installed stay put until you upgrade them. Bundled charts (Canal, CoreDNS, ingress-nginx when enabled) move with the RKE2 version. |
