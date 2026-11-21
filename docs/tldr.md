# Install and upgrade, TLDR

The command, then the object it writes. Flags, channels, and what a CVE actually moves are in the linked page. Versions below match the rest of this repo (November 2026). Dry-run before you apply.

| | Install | Upgrade Kubernetes | OS only, same Kubernetes | Vendor CSI / CNI only | Object to watch |
| --- | --- | --- | --- | --- | --- |
| [OpenShift](platforms/openshift.md) | `openshift-install create cluster --dir=ocp` | `oc adm upgrade --to=4.y.z` | No separate command. The z-stream is both. | Payload: the same upgrade. Your operator: approve the `InstallPlan`. | `clusterversion`, `mcp` |
| [NKP](platforms/nkp.md) | `nkp create cluster <provider>` | `nkp upgrade cluster nutanix --vm-image <image>` | Only if a second image exists for that same Kubernetes version. Still new VMs. | `nkp upgrade addons <provider> --cluster-name` | `machines`, `nkpclusters` |
| [GKE](platforms/gke.md) | `gcloud container clusters create` | `gcloud container clusters upgrade --master`, then `--node-pool` | Newer `-gke.N` on the pool. No minor bump. | No. They ride the GKE version. Your chart: `helm upgrade`. | `gcloud container operations list` |
| [EKS](platforms/eks.md) | `aws eks create-cluster`, then node group, then add-ons | `aws eks update-cluster-version` | `update-nodegroup-version` with a new AMI, same Kubernetes version | `aws eks update-addon` | `aws eks describe-update`, `describe-addon` |
| [EKS Anywhere](platforms/eks-anywhere.md) | `eksctl anywhere create cluster -f cluster.yaml` | `eksctl anywhere upgrade cluster -f cluster.yaml` | New template or `osImageURL` at the same `kubernetesVersion` | CNI: no, it is in the cluster upgrade. CSI: `helm upgrade`. | `clusters.anywhere.eks.amazonaws.com`, CAPI `machines` |
| [AKS](platforms/aks.md) | `az aks create` | `az aks upgrade -k <version>` | `az aks upgrade --node-image-only` | No. They ride the AKS version. | `az aks show`, node image label |
| [RKE2](platforms/rke2-rancher.md) | `install.sh` then `systemctl enable --now rke2-server` | New version, restart `rke2-server`, then agents | OS package manager. RKE2 version stays. | Bundled CNI: the RKE2 bump. Your chart: `helm upgrade`. | `kubectl get nodes` |

## OpenShift

Channels and install-config fields: [OpenShift](platforms/openshift.md).

```bash
# install
openshift-install create install-config --dir=ocp
openshift-install create cluster --dir=ocp
export KUBECONFIG=ocp/auth/kubeconfig

# what will I be offered
oc adm upgrade
oc adm upgrade channel stable-4.19          # or fast-4.19, eus-4.18, candidate-4.19

# upgrade. pause workers first if masters should move alone
oc patch mcp worker --type merge -p '{"spec":{"paused":true}}'
oc adm upgrade --to=4.19.6
oc patch mcp worker --type merge -p '{"spec":{"paused":false}}'

# disconnected, no update service
oc adm upgrade --allow-explicit-upgrade --to-image registry.site.example:5000/openshift/release@sha256:DIGEST

# air-gap mirror
oc-mirror --config imageset-config.yaml docker://registry.site.example:5000 --v2

# an operator you subscribed to, not the payload
oc get installplan -n openshift-cluster-csi-drivers
oc patch installplan <name> -n openshift-cluster-csi-drivers --type merge -p '{"spec":{"approved":true}}'
```

| Kind | Name people use | What an install or upgrade writes |
| --- | --- | --- |
| `ClusterVersion` | `version` | `spec.channel`, `spec.desiredUpdate`, `spec.upstream` (disconnected graph). This is the upgrade. |
| `ClusterOperator` | `oc get co` | One row per payload operator. `network`, `etcd`, `machine-config` degraded means the upgrade has not finished. |
| `Network` | `network.operator.openshift.io/cluster` | OVN-Kubernetes config. Many fields render a `MachineConfig` and reboot nodes. |
| `MachineConfigPool` | `master`, `worker` | `spec.paused`, `spec.maxUnavailable`. `UPDATING=True` is the node roll. |
| `MachineConfig` | `99-<role>-…` | Node OS content (kernel arg, file, systemd). Applying one reboots that pool. |
| `ImageDigestMirrorSet` | from `oc-mirror` | Registry rewrite for a disconnected cluster. |
| `UpdateService` | in `openshift-update-service` | In-site Cincinnati. Point `ClusterVersion.spec.upstream` at `status.policyEngineURI`. |
| `Subscription` | per operator namespace | Channel of an OLM operator. |
| `InstallPlan` | per operator namespace | The thing you approve to move that operator without moving the payload. |
| `ClusterServiceVersion` | per operator namespace | The operator version now running. |
| `StorageClass` | — | Speed 1. Does not upgrade a driver. |

`oc get clusterversion` and `oc get mcp` are the two watches. Operators first, then pools.

## NKP

Providers: `nutanix`, `preprovisioned`, `aws`, `azure`, `vsphere`, `gcp`, `eks`. Order is Kommander, workspaces, then the cluster. One NKP minor at a time. Management cluster first.

```bash
# install, Nutanix
nkp create cluster nutanix --cluster-name "$CLUSTER_NAME" --self-managed \
  --control-plane-vm-image "$NKP_IMAGE" --worker-vm-image "$NKP_IMAGE" \
  --csi-storage-container "$STORAGE_CONTAINER"
# air gap: add --airgapped and either --registry-mirror-* or --bundle, not both
# docker load the konvoy bootstrap image first

nkp get kubeconfig --cluster-name "$CLUSTER_NAME" > "$CLUSTER_NAME.conf"

# upgrade
nkp upgrade kommander
# air gap, from the bundle root:
nkp upgrade kommander --kommander-applications-repository ./application-repositories/kommander-applications-vX.Y.Z.tar.gz
nkp upgrade workspace "$WORKSPACE_NAME"
nkp upgrade cluster nutanix --cluster-name "$CLUSTER_NAME" -n "$NS" --vm-image "$VM_IMAGE_NAME"

# other providers, split
nkp upgrade capi-components
nkp upgrade addons aws --cluster-name "$CLUSTER_NAME"
nkp upgrade cluster eks --cluster-name "$CLUSTER_NAME"

# see the CNI/CSI manifests before they overwrite your edits
nkp upgrade addons aws --cluster-name "$CLUSTER_NAME" --dry-run -o yaml
```

| Kind | What it is |
| --- | --- |
| `NKPCluster` | NKP's cluster object. `kubectl get nkpclusters -A`. The management one is labeled `kommander.d2iq.io/host=true`. |
| `Cluster` | `cluster.cluster.x-k8s.io`. CAPI's cluster. |
| `KubeadmControlPlane` | `spec.version` is the control-plane Kubernetes version. `infrastructureTemplate` points at the image template. |
| `MachineDeployment` | Worker pool. `spec.template.spec.version` plus `infrastructureRef`. |
| `Machine` | One VM. `Provisioning` then `Running` then `Deleting` is a healthy roll. |
| `NutanixMachineTemplate` | Immutable. New image means a new template name. Editing the old one does not roll. |
| `ClusterResourceSet` | Bundled CNI, CSI, autoscaler, NFD. `nkp upgrade addons` rewrites these. Hand edits die. |
| `ClusterClass` | NKP 2.19 topology. Edit the `NKPCluster`, not the generated MachineDeployment. |
| `HelmRelease` | `helm.toolkit.fluxcd.io`. Platform apps and catalog apps Kommander deployed. |
| `AppDeployment` | One platform app or one catalog app. Platform `AppDeployment`s bump as a set. Catalog ones bump one at a time. |

Annotation `topology.cluster.x-k8s.io/upgrade-concurrency` on the CAPI `Cluster` is how many worker pools roll at once. Default 1.

## GKE

Channel values and every create flag: [GKE](platforms/gke.md). No CRD owns the cluster. The GKE API does. `kubectl` sees the result.

```bash
gcloud container get-server-config --location "$REGION" --format='yaml(channels)'

gcloud container clusters create "$CLUSTER" --location "$REGION" \
  --release-channel regular --enable-dataplane-v2 --enable-ip-alias \
  --machine-type n2-standard-8 --num-nodes 3 \
  --workload-pool "$PROJECT.svc.id.goog"
# Autopilot: gcloud container clusters create-auto "$CLUSTER" --location "$REGION" --release-channel regular
# --num-nodes is per zone. A region with 3 zones and --num-nodes 3 is 9 VMs.

# control plane only
gcloud container clusters upgrade "$CLUSTER" --master --cluster-version "$VERSION" --location "$REGION"
# nodes
gcloud container clusters upgrade "$CLUSTER" --node-pool "$POOL" --cluster-version "$VERSION" --location "$REGION"

# one add-on, not --addons on create (that replaces the whole set)
gcloud container clusters update "$CLUSTER" --location "$REGION" \
  --update-addons GcpFilestoreCsiDriver=ENABLED
```

| Object | Where |
| --- | --- |
| Cluster, release channel, node pool | GKE API. `gcloud container clusters describe`, `operations list`. |
| `StorageClass` | `standard-rwo`, `premium-rwo`. Adding one does not change the CSI version. |
| `UserClusterMetadata` | GDC air-gapped. Lists Kubernetes versions the appliance already has. |
| `UserClusterUpgradeRequest` | `cluster.gdc.goog/v1`. The only upgrade API on GDC. `targetVersion` must be one of those listed. |

## EKS

Add-ons do not move when the cluster version moves.

```bash
aws eks create-cluster --name "$CLUSTER" --kubernetes-version 1.32 --role-arn "$CLUSTER_ROLE_ARN" \
  --resources-vpc-config subnetIds="$SUBNETS"
aws eks create-nodegroup --cluster-name "$CLUSTER" --nodegroup-name general \
  --subnets $SUBNETS --node-role "$NODE_ROLE_ARN" --update-config maxUnavailable=1
aws eks create-addon --cluster-name "$CLUSTER" --addon-name vpc-cni --addon-version "$V"
aws eks create-addon --cluster-name "$CLUSTER" --addon-name aws-ebs-csi-driver \
  --addon-version "$V" --service-account-role-arn "$EBS_ROLE"

aws eks describe-addon-versions --addon-name vpc-cni --kubernetes-version 1.32
aws eks describe-addon-configuration --addon-name vpc-cni --addon-version "$V"

aws eks update-cluster-version --name "$CLUSTER" --kubernetes-version 1.33
aws eks update-addon --cluster-name "$CLUSTER" --addon-name vpc-cni \
  --addon-version "$V" --resolve-conflicts PRESERVE
aws eks update-nodegroup-version --cluster-name "$CLUSTER" --nodegroup-name general
```

| API | What it versions |
| --- | --- |
| `Cluster` | Control plane Kubernetes version. |
| `Nodegroup` | AMI and kubelet. `update-nodegroup-version` without a cluster-version change is the OS CVE at the same Kubernetes version. |
| `Addon` | `vpc-cni`, `kube-proxy`, `coredns`, `aws-ebs-csi-driver`, `aws-efs-csi-driver`. Own version, own `configurationValues`. |
| In-cluster | The add-on writes a DaemonSet or Deployment. `resolve-conflicts PRESERVE` keeps kubectl edits. `configuration-values` replaces the whole blob. |

No cluster CRD. EKS Anywhere is the product that has one.

## EKS Anywhere

```bash
eksctl anywhere generate clusterconfig "$CLUSTER" --provider vsphere > cluster.yaml
eksctl anywhere create cluster -f cluster.yaml

# air gap, on the admin machine, before create or upgrade
eksctl anywhere download artifacts
eksctl anywhere download images -o images.tar
eksctl anywhere import images -i images.tar -r "$REGISTRY" \
  --bundles ./eks-anywhere-downloads/bundle-release.yaml

eksctl anywhere upgrade plan cluster -f cluster.yaml
eksctl anywhere upgrade cluster -f cluster.yaml
# workload cluster can also be kubectl-applied to the management cluster
```

| Kind | API | What an upgrade changes |
| --- | --- | --- |
| `Cluster` | `anywhere.eks.amazonaws.com/v1alpha1` | `spec.kubernetesVersion`, `spec.eksaVersion`, `spec.clusterNetwork.cniConfig` (immutable after create), `registryMirrorConfiguration`, `upgradeRolloutStrategy`. |
| Provider configs | `VSphereMachineConfig`, Nutanix, Tinkerbell, Snow | The OS template. Bare metal: `osImageURL`, `hookImagesURLPath`. |
| CAPI `Machine` | `cluster.x-k8s.io` | The roll. Control plane machines first, then workers. |
| Package objects | curated packages | Not in `upgrade cluster`. Separate upgrade, separate image import. |
| Your CSI Helm release | — | Not in `upgrade cluster`. |

## AKS

```bash
az aks create -g "$RG" -n "$CLUSTER" -k 1.32.0 \
  --network-plugin azure --network-plugin-mode overlay --network-dataplane cilium \
  --auto-upgrade-channel stable --node-os-upgrade-channel NodeImage

az aks get-upgrades -g "$RG" -n "$CLUSTER" -o table

# control plane only. not legal while cluster autoupgrade is on
az aks upgrade -g "$RG" -n "$CLUSTER" -k 1.32.3 --control-plane-only
# one pool
az aks nodepool upgrade -g "$RG" --cluster-name "$CLUSTER" -n nodepool1 -k 1.32.3
# OS image, same Kubernetes version
az aks nodepool upgrade -g "$RG" --cluster-name "$CLUSTER" -n nodepool1 --node-image-only
# both
az aks upgrade -g "$RG" -n "$CLUSTER" -k 1.32.3 --max-surge 1

# Cilium data plane. reimages every pool. not a version bump
az aks update -g "$RG" -n "$CLUSTER" --network-dataplane cilium

# many clusters
az fleet updaterun create -g "$RG" --fleet-name "$FLEET" --name run1 \
  --upgrade-type Full --kubernetes-version 1.32.3 --node-image-selection Latest \
  --stages @stages.json
# --upgrade-type NodeImageOnly for the OS week
```

| Object | Where |
| --- | --- |
| Managed cluster | `kubernetesVersion` is the control plane. |
| Agent pool | `orchestratorVersion` is the kubelet. `nodeImageVersion` is the OS. `--node-image-only` changes the second. |
| Fleet update run | Stages and groups. `Full` moves Kubernetes and the image. `NodeImageOnly` does not. |
| `StorageClass` | `disk.csi.azure.com`. Does not upgrade the driver. |

`--auto-upgrade-channel`: `none`, `patch`, `stable`, `rapid`, `node-image`. `--node-os-upgrade-channel`: `None`, `Unmanaged`, `SecurityPatch`, `NodeImage`.

## RKE2 and Rancher

```bash
# first server
sudo INSTALL_RKE2_VERSION=v1.32.3+rke2r1 sh install.sh
sudo systemctl enable --now rke2-server
# air gap: copy the images tarball to /var/lib/rancher/rke2/agent/images/
# and INSTALL_RKE2_ARTIFACT_PATH=/root/rke2

# upgrade, one server at a time, then agents
sudo INSTALL_RKE2_VERSION=v1.32.4+rke2r1 sh install.sh
sudo systemctl restart rke2-server
sudo systemctl restart rke2-agent
```

`/etc/rancher/rke2/config.yaml` is the install file: `cni`, `system-default-registry`, `cluster-cidr`, `service-cidr`, `token`, `server`. `/etc/rancher/rke2/registries.yaml` is the mirror CA.

| Kind | What it is |
| --- | --- |
| `HelmChart` | `helm.cattle.io`. RKE2 applies these from `/var/lib/rancher/rke2/server/manifests`. A file RKE2 shipped gets replaced on upgrade. Put your chart in your own file. |
| `Cluster` | `management.cattle.io`, on the Rancher management cluster. Downstream Kubernetes version and upgrade concurrency. |
| `GitRepo` | `fleet.cattle.io`. App rollouts across clusters. Not the RKE2 version. |

## Same watches, every cluster

```bash
kubectl get nodes
kubectl get csidriver
kubectl get storageclass
kubectl get ds -A | grep -E 'cilium|calico|aws-node|ovn|csi'
```

Plus the platform object from the table at the top. A new `StorageClass` never means the driver was upgraded.

## Logging, monitoring, platform apps

The long version is [Platform apps](platform-apps.md). NKP is the only one that upgrades the default set because the platform version moved.

```bash
# NKP, before the Kubernetes upgrade. 2.19: nkp upgrade platform-apps
nkp upgrade kommander
nkp upgrade workspace "$WORKSPACE_NAME"
# a catalog app is not in that set
nkp create appdeployment <app> --workspace "$WORKSPACE_NAME"

# OpenShift. monitoring is already in the payload. logging is a subscription.
oc get co monitoring
oc get subscription,installplan,csv -A

# GKE. collectors are not charts. they move with the node.
gcloud container clusters update "$CLUSTER" --location "$REGION" \
  --logging=SYSTEM,WORKLOAD --monitoring=SYSTEM

# EKS. not installed by default, and update-cluster-version does not bump it
aws eks update-addon --cluster-name "$CLUSTER" \
  --addon-name amazon-cloudwatch-observability --addon-version "$V" \
  --resolve-conflicts PRESERVE

# AKS. enablement, not a chart pin
az aks update -g "$RG" -n "$CLUSTER" --enable-azure-monitor-metrics \
  --azure-monitor-workspace-resource-id "$WORKSPACE_ID"

# EKS Anywhere. upgrade cluster does not change this
kubectl get packages -A

# Rancher. downstream k8s upgrade does not upgrade these charts
# Apps & Marketplace → Installed Apps → rancher-monitoring, rancher-logging
```
