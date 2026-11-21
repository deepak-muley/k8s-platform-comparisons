# Comparison

Read [concepts](concepts.md) for the speed numbers. Speed 1 is a config write. Speed 2 is a Deployment or DaemonSet roll. Speed 3 is a node drain and replace. Speed 4 is a new cluster.

Versions below are the products as documented in November 2026. CNI defaults have moved recently on NKP (Cilium on Nutanix infrastructure, Calico still generated in the AWS/pre-provisioned upgrade examples). Trust a dry-run of the release you are installing.

## Who owns what

| | OpenShift | NKP | GKE | EKS | EKS Anywhere | AKS | RKE2 + Rancher |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Control plane | You, on masters. CVO reconciles it from a release payload. | You, on VMs or pre-provisioned hosts. CAPI replaces them. | Google | AWS | You. CAPI replaces machines. | Azure | You. `rke2-server` on the nodes. |
| Node image | RHCOS, rendered by the Machine Config Operator | Vendor image (NKP Ubuntu/RHEL image in Prism, AMI, Azure image) or your pre-provisioned OS | Container-Optimized OS or Ubuntu, Google-built | EKS-optimized AMI, or your AMI, or Auto Mode | Bottlerocket / Ubuntu images EKS Anywhere publishes, or the OS you supply on bare metal | Azure node image, separate from the Kubernetes version | The OS you installed. RKE2 is a package on top. |
| Default CNI | OVN-Kubernetes, in the release payload | Cilium on current Nutanix-infrastructure clusters. Calico (Tigera operator) is what other providers' generated add-on still shows. Check the dry-run. | Dataplane V2 (Cilium-based) or kube-proxy. Google-managed. | Amazon VPC CNI, as a managed add-on | Cilium, or kindnet for non-production. Fixed after create. | Azure CNI Overlay, or Azure CNI powered by Cilium | Canal (Calico + Flannel), or Cilium/Calico if you set `cni:` |
| CSI the installer gives you | Cloud CSI operators shipped or subscribed for that platform. Cluster Storage Operator is in the payload. | Core add-on via `ClusterResourceSet` (EBS, Azure Disk, vSphere, Nutanix). Pre-provisioned Nutanix CSI is often a Helm install you do yourself. | GCE PD CSI, on by default. Filestore and Cloud Storage FUSE are toggleable managed add-ons. | EBS CSI is a managed add-on you create. It is not implicit. | None since EKS Anywhere v0.17. You install vSphere CSI or another driver. | Azure Disk, File, and Blob CSI, managed with the cluster. | None. You install one. |
| Fleet product | Advanced Cluster Management, or hosted control planes (HyperShift) | Kommander workspaces and `NKPCluster` | Fleets, release channels | None built in. You write the loop. | Management cluster plus workload `Cluster` objects | Azure Kubernetes Fleet Manager | Rancher Manager and Fleet |

## Kubernetes version vs the OS image

The full rules, the CAPX objects, and what a replacement throws away are in [Cluster API](capi.md). This is the row to answer "can we upgrade Kubernetes and leave the OS."

| | What a node upgrade does | Kubernetes without a new OS image | OS image without a new Kubernetes version |
| --- | --- | --- | --- |
| OpenShift | Machine Config Operator rebases the existing VM and reboots it. etcd data stays. | No. The z-stream ships RHCOS and the kubelet together. | No separate OS channel. That z-stream is the OS update. |
| NKP on Nutanix (CAPX) | New VM from `NutanixMachineTemplate`, then the old VM is deleted. | No. The Prism image must match that Kubernetes version. `imageLookup` selects the image from the version. | Only if a second image exists for the same Kubernetes version. Still a new VM. No in-place `dnf`. |
| NKP pre-provisioned (CAPPP) | The host stays. Cluster API does not clone it. | Yes. Bootstrap upgrades Kubernetes on that host. | Yes. You patch the OS. Cluster API will not. |
| NKP on AWS, Azure, vSphere, GCP | New instance from that provider's template. | No for the Konvoy image, which is built per Kubernetes version. | New template at the same version, if that image exists. |
| EKS Anywhere | Cluster API replacement (CAPV, CAPX, Tinkerbell, Snow). | The published image is per `kubernetesVersion`. | New template or `osImageURL` at the same version. Bare metal needs a spare host to surge. |
| GKE | New GCE VM. | No. `1.32.8-gke.1056000` is the image and the kubelet. | Yes, a newer `-gke.N` build inside the minor. The VM is still new. |
| EKS | Control plane is updated in AWS. Nodes are new EC2 instances. | Control plane yes. Nodes no. The EKS AMI matches one Kubernetes version. | Yes. New AMI, same Kubernetes version, `update-nodegroup-version`. |
| AKS | New VM via surge. | No. `-k` moves Kubernetes and the node image together. | Yes. `--node-image-only` or Fleet `NodeImageOnly`. |
| RKE2 | Same host, systemd restart. | Yes. | Yes. The OS package manager, then a reboot if the kernel changed. |

Finer than that — a CVE in CSI only, CNI only, or the control plane while nodes stay — is [CVE scope](cve-scope.md). OpenShift channels and the install-config fields are on the [OpenShift](platforms/openshift.md) page, in the same shape as the GKE channel and flag tables.

## How a CSI or CNI change actually lands

| Change | OpenShift | NKP | GKE | EKS | EKS Anywhere | AKS | RKE2 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Bump the vendor CNI version | With the release payload. Speed 3, because the Machine Config Operator rolls nodes for the OS content that ships with the payload. The OVN pods also roll (speed 2) inside that upgrade. | `nkp upgrade addons <provider>` or inside `nkp upgrade cluster nutanix`. DaemonSet roll (speed 2) unless the cluster upgrade also replaces VMs (speed 3). | You do not. Google ships it with the node and control plane. Speed 3 when nodes upgrade. | `aws eks update-addon --addon-name vpc-cni`. Speed 2. | Inside `eksctl anywhere upgrade cluster`. Speed 3. | Inside `az aks upgrade`, or the Cilium data-plane migration, which reimages all pools. Speed 3. | Inside the RKE2 version bump. Speed 3. |
| Change CNI config (MTU, policy engine, IPAM knobs) | `network.operator` / `Network` CR. Many fields roll every node. Speed 3. | Helm values inside the generated add-on. Custom values are wiped by `nkp upgrade addons` unless you reapply them from a dry-run. Speed 2 for a DaemonSet env change. | Cluster update API. Dataplane features that touch the node reimage nodes. | `configurationValues` on the add-on, schema from `describe-addon-configuration`. Speed 2. | Cilium Helm values are not a supported free-form field. Treat config as part of the cluster upgrade. | Limited. Data-plane and IPAM mode changes reimage nodes. | Edit the HelmChart config for the CNI chart, or `cni` config. DaemonSet roll. Switching `cni:` after install is a rebuild. |
| Replace the CNI with a different project | Rebuild. OVN-Kubernetes is the cluster network. | At create only: `--dry-run`, delete the CNI `ClusterResourceSet`, install the other CNI before nodes go Ready. | Create-time only. `--enable-dataplane-v2` cannot be added to an existing Standard cluster. Autopilot is already on it. | Not the normal path. VPC CNI is the supported primary CNI. | Rebuild. `cniConfig` is immutable. | Supported migration to Azure CNI powered by Cilium. Reimages every node pool in one operation. | Install with `cni: none` and bring your own, or rebuild. |
| Bump the vendor CSI version | If it is a payload or release-catalog operator, it moves with the minor or with a new catalog index. Speed 2 for the CSI pods. Third-party CSI is `helm upgrade`, speed 2, any day. | Bundled CSI: `nkp upgrade addons`. Speed 2. Self-managed Nutanix CSI on pre-provisioned: `helm upgrade`, any day. | Google moves GCE PD CSI with GKE. You enable or disable. Third-party CSI is yours, speed 2. | `aws eks update-addon --addon-name aws-ebs-csi-driver`. Speed 2. Independent of the node AMI. | `helm upgrade` whenever you want. Speed 2. You own compatibility. | Moves with the AKS version. Speed 2 inside a cluster upgrade. Node-image-only does not bump the CSI chart. | `helm upgrade` whenever you want. Speed 2. |
| Change a StorageClass | Speed 1. A new `StorageClass` object. Existing PVs keep the class they were created with. | Same. | Same. | Same. | Same. | Same. | Same. |
| Change the default StorageClass | Speed 1. Annotation flip. Does not restripe existing volumes. | Same. On NKP pre-provisioned, the local-volume class is often still default until you flip it. | Same. | Same. | Same. | Same. | Same. |

## Install, connected vs air-gapped

| | Connected install | Air-gapped install | What you transfer |
| --- | --- | --- | --- |
| OpenShift | `openshift-install create cluster` (IPI), Assisted Installer, or Agent-based. Release image pulled from `quay.io`. | `oc-mirror` into an internal registry. `ImageDigestMirrorSet` + release signatures. Install from the mirrored release image. | Release payload, operator catalogs, samples. Often 30 GB+ per minor. |
| NKP | `nkp create cluster <provider>` from a bastion with internet. Bootstrap is a temporary kind cluster. | `--airgapped` plus either `--registry-mirror-*` or `--bundle`. Do not pass both. `docker load` the bootstrap image first. | `nkp-air-gapped-bundle` (~15 GB for 2.17): Konvoy images, Kommander images, charts, CLI. |
| GKE | `gcloud container clusters create` or Autopilot. | No air gap. Use Google Distributed Cloud air-gapped. Clusters are created against a local management API. | Google ships the GDC appliance. You do not mirror GKE public images yourself. |
| EKS | `aws eks create-cluster`, then a node group or Auto Mode, then add-ons. | No air gap for regional EKS. Use EKS Anywhere. | Nothing. The control plane is in the region. |
| EKS Anywhere | `eksctl anywhere create cluster -f cluster.yaml` from an admin machine. | `download artifacts`, `download images`, `import images` into a registry mirror. Set `registryMirrorConfiguration`. Bare metal also needs `osImageURL` and `hookImagesURLPath`. | Images tarball plus the downloads bundle. Admin machine needs on the order of 80 GB free while importing. |
| AKS | `az aks create`. | No air gap for public AKS. Azure Local / disconnected marketplace is the counterpart, and it is a different product surface. | Node images and Azure Local artifacts, staged ahead of the site. |
| RKE2 | `curl install.sh \| sh`, or the Rancher provisioning flow. | Copy the RKE2 air-gap images tarball and the install script. `INSTALL_RKE2_ARTIFACT_PATH`. Set `system-default-registry`. | One images tarball per RKE2 version, plus the binary/script. Rancher air-gap is a second, larger set. |

## Upgrade order

| | First | Then | Then | CNI/CSI bundled into the cluster upgrade? |
| --- | --- | --- | --- | --- |
| OpenShift | Mirror the next release and update operator catalogs if disconnected. Update OLM operators so a CSV exists for the target minor. | `oc adm upgrade`. CVO moves cluster operators. | Machine Config Operator rolls masters, then workers. | Yes for payload components. Optional operators only if you already moved them. |
| NKP | New CLI. New VM image in Prism (Nutanix) or new artifacts in the mirror. `nkp upgrade kommander`, then `nkp upgrade workspace` so platform apps understand the next Kubernetes APIs. | Management cluster, then managed clusters. On non-Nutanix: `nkp upgrade capi-components`, `nkp upgrade addons`, then the Kubernetes version. On Nutanix: `nkp upgrade cluster nutanix --vm-image`. | Worker pools. Default concurrency is 1 pool at a time. Annotation `topology.cluster.x-k8s.io/upgrade-concurrency` raises it. | On Nutanix, `nkp upgrade cluster nutanix` upgrades CNI and CSI as part of the cluster upgrade. On other providers, core add-ons are their own command and must be done before or with the documented sequence, not forgotten after. |
| GKE | Pick a version from `gcloud container get-server-config` that the channel offers. | Control plane. | Node pools, surge or blue-green. Autopilot does both. | Yes. You do not run a separate CNI/CSI upgrade for Google-managed drivers. |
| EKS | Read upgrade insights. Confirm add-on compatibility for the target minor. | `aws eks update-cluster-version`. | Managed node group or Karpenter drift. Then `aws eks update-addon` for vpc-cni, coredns, kube-proxy, EBS CSI. Those do not move by themselves. | No. Add-ons are a separate step. This is the easiest product to leave on an old CNI by accident. |
| EKS Anywhere | Upgrade the CLI. Import the new images into the mirror if air-gapped. `upgrade plan`. | One minor. Management components, then cluster components. `maxSurge` / `maxUnavailable`. | Workload clusters by applying their spec (kubectl, GitOps, Terraform). | CNI yes. CSI no. |
| AKS | Fleet strategy if you have many clusters. Otherwise one cluster. | Control plane, then node pools. Or `NodeImageOnly` when you only need the OS. | `--network-dataplane cilium` is its own operation and reimages every pool. | CSI yes, as part of the AKS version. Node image no. |
| RKE2 | Stage the new air-gap tarball. Upgrade Rancher first if it manages the cluster. | RKE2 servers, one at a time. Wait until the server is ready. | Agents. Bundled charts move with the version. | Bundled CNI yes. Your CSI chart no. |

## Scale mechanism

| | Unit of rollout | Canary | What stops a bad rollout |
| --- | --- | --- | --- |
| OpenShift | One cluster, unless ACM or a hosted-control-plane fleet applies the `ClusterVersion` change. | Pause the worker `MachineConfigPool`. A custom pool can take one node first. | Paused pool. `maxUnavailable` on the pool. CVO will not skip a failing cluster operator. |
| NKP | One `NKPCluster`. A workspace upgrade touches every cluster in that workspace for platform apps. | Upgrade a non-prod workspace, then prod. Inside a cluster, one node pool at a time unless you raise concurrency. | Preflight checks on `nkp upgrade cluster`. PodDisruptionBudgets. Failed Machine objects stay visible in CAPI. |
| GKE | Release channel membership is already a fleet. Maintenance windows stagger it. | A cluster on Rapid, then Regular, then Stable. Or a single node pool with surge 1. | Maintenance exclusions. Soak time on the slower channel. Surge `maxUnavailable=0` keeps capacity. |
| EKS | One cluster API call. You supply the fan-out. | One cluster, one node group, `maxUnavailable` 1. | Upgrade insights (pre-check). CloudTrail and the update status. Nothing automatic stops the next cluster, because nothing automatic started it. |
| EKS Anywhere | One `Cluster` object. | One workload cluster. `maxUnavailable: 0` and `maxSurge: 1`. | Preflight in `upgrade cluster`. CAPI Machine health. |
| AKS | A Fleet Manager update run: stages, then groups inside a stage. | Stage 1 is test, stage 2 waits, stage 3 is prod. `afterStageWaitInSeconds`. Gates via approvals. | `maxAllowedFailures`. Member maintenance windows. A failed stage stops the run when the failure budget is 0. |
| RKE2 / Rancher | Rancher cluster object, or a Fleet `GitRepo` for apps. | A Fleet bundle target that matches only the canary cluster. Downstream cluster concurrency in Rancher. | Drain timeout. A node that does not come back Ready blocks the rest if you upgrade servers strictly one by one. |

## Fastest supported way to change each thing

This is the column to use when someone asks "can we just patch it".

| I need to… | Fastest supported path | Speed |
| --- | --- | --- |
| Ship a CVE fix in the node OS without moving Kubernetes | GKE: node pool upgrade stays on the same minor if a newer image exists. AKS: `--node-image-only`. EKS: new AMI on the node group, same Kubernetes version. OpenShift: a new RHCOS boot image only arrives with a z-stream, so the z-stream is the path. NKP: a new VM image at the same Kubernetes version if the release provides one; otherwise it is a full cluster upgrade. RKE2: patch the OS with the OS package manager, then restart `rke2-server` / `rke2-agent` if the kernel changed. | 3, but you avoid a Kubernetes minor |
| Move VPC CNI or EBS CSI this afternoon | EKS `update-addon` | 2 |
| Move Azure Disk CSI alone | You don't. It tracks AKS. Move a third-party CSI with Helm if you need a same-day bump. | 2 only for third-party |
| Move GCE PD CSI alone | You don't. | — |
| Move Nutanix CSI on an NKP pre-provisioned cluster | `helm upgrade` the Nutanix CSI chart you installed | 2 |
| Move Nutanix CSI on an NKP AHV cluster that NKP installed | `nkp upgrade addons` / cluster upgrade, bundled version only | 2 inside a platform upgrade |
| Turn on a new StorageClass | `kubectl apply` a StorageClass. Same everywhere. | 1 |
| Change pod CIDR | New cluster. Same everywhere. | 4 |
| Put Cilium in front of an existing cluster | AKS data-plane update (reimage). NKP, EKS Anywhere, OpenShift, and GKE Standard: only if the cluster was created that way. GKE cannot add Dataplane V2 later. | 3 or 4 |
