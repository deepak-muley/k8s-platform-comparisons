# Azure Kubernetes Service

AKS is a managed control plane and managed node images. Azure CNI is the network, and Azure Disk, File, and Blob CSI are part of the cluster. You do not helm-upgrade those the way you helm-upgrade a third-party driver. You move AKS, and they move. The fast *OS* path is separate: `--node-image-only` replaces node images without moving the Kubernetes minor.

Public AKS is not an air gap. Azure Local is the on-prem surface, and its disconnected story is a different runbook. Fleet Manager is the at-scale upgrade product, and it is the one in this repo that has stages, groups, and a failure budget as first-class API fields.

AKS is not Cluster API, and it still replaces the VM. `az aks upgrade -k` moves the Kubernetes version and the node image together, which is the CAPX-shaped restriction. `--node-image-only` (and Fleet `--upgrade-type NodeImageOnly`) is the operation CAPX does not have: a new OS image, the same Kubernetes version, nodes still surged rather than patched in place. See [Cluster API](../capi.md).

## Core vs add-on

| | What | How you change it | Speed |
| --- | --- | --- | --- |
| Core | Control plane | `az aks upgrade -k <version>` or a Fleet update run of type `Full` | API operation, then AKS rolls the control plane |
| Core | Node image and kubelet | The same upgrade, or `--node-image-only` | 3, with `--max-surge` |
| Platform core | Azure CNI (node subnet, overlay, or dynamic IP allocation) and the data plane (Azure NPM, or Cilium) | Create-time, plus a supported migration (`az aks update --network-dataplane cilium`, or an IPAM mode migration) | 3. The Cilium data-plane update reimages every node pool. You cannot do it one pool at a time. |
| Platform core | Azure Disk CSI, Azure File CSI, Azure Blob CSI | They track the AKS version | 2 inside an AKS upgrade. No independent add-on version pin like EKS. |
| Add-on | Anything you Helm-installed, including a third-party CSI | `helm upgrade` | 2 |
| Fleet-only | Cross-cluster Cilium components, when you opted into Fleet cross-cluster networking | Fleet, bundled with AKS Kubernetes releases | You upgrade member control planes with an update run. You do not upgrade that Cilium mesh by hand. |

Azure CNI powered by Cilium is the data plane Azure wants clusters on. It can be selected at create, or applied later. The later path is a reimage of all pools, and it is unsupported on clusters with Windows node pools. Clusters using node auto-provisioning need that turned off for the duration of the data-plane update.

Kubenet is legacy. A kubenet cluster does not jump straight to the Cilium data plane. Migrate the IPAM mode to Azure CNI Overlay first, then update the data plane. Those are two operations, in either order when you are already on Azure CNI. They are not one operation.

## Create

```bash
az aks create \
  --resource-group "${RG}" \
  --name "${CLUSTER}" \
  --kubernetes-version 1.32.0 \
  --location "${LOCATION}" \
  --node-count 3 \
  --node-vm-size Standard_D8s_v5 \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --network-dataplane cilium \
  --max-surge 1 \
  --auto-upgrade-channel stable \
  --node-os-upgrade-channel NodeImage

az aks get-credentials -g "${RG}" -n "${CLUSTER}"
kubectl get nodes
kubectl get storageclass
kubectl get ds -n kube-system
```

`--auto-upgrade-channel` is the cluster minor/patch cadence (`none`, `patch`, `stable`, `rapid`, `node-image`). `--node-os-upgrade-channel` is the OS image cadence. Set them on purpose. `none` means you run every upgrade yourself, including the one AKS will otherwise do when your minor falls out of support.

Overlay mode gives pods IPs from a range that is not the subnet's, which is what makes large clusters fit in a VNet. It is the right default. Switching IPAM mode later is possible and is a migration, not a flag you flip casually. Pod CIDR planning still happens at create.

## Day-2

Node pool that is a different size, a different subnet, or a different OS. Pools are the unit. Immutable fields mean a new pool.

```bash
az aks nodepool add \
  -g "${RG}" --cluster-name "${CLUSTER}" \
  -n mem \
  --node-count 3 \
  --node-vm-size Standard_E8s_v5 \
  --max-surge 1 \
  --mode User

kubectl cordon -l kubernetes.azure.com/agentpool=system
# drain workloads that can move, then delete the old pool when it is empty
```

Maintenance window, so auto-upgrade and Fleet runs have a legal time:

```bash
az aks maintenanceconfiguration add \
  -g "${RG}" --cluster-name "${CLUSTER}" \
  -n aksManagedAutoUpgradeSchedule \
  --schedule-type Weekly \
  --day-of-week Tuesday --start-time 00:00 --interval-hours 4
```

A StorageClass is a StorageClass. Azure Disk's provisioner is `disk.csi.azure.com`. `managed-csi` is the usual default. Add `managed-csi-premium` or your own class without touching the driver. Expansion and snapshots depend on the class and on the snapshot CRDs AKS installs. Test them.

## Upgrade one cluster

```bash
az aks get-upgrades -g "${RG}" -n "${CLUSTER}" -o table

az aks upgrade \
  -g "${RG}" -n "${CLUSTER}" \
  -k 1.32.3 \
  --max-surge 1 \
  --yes
```

Control plane first, then each node pool, surge at a time. `--max-surge 1` needs one spare VM's worth of quota and one spare IP per pool. A surge that cannot start looks like a stuck upgrade.

Node image only, the week you are patching Ubuntu/Azure Linux and not Kubernetes:

```bash
az aks nodepool upgrade \
  -g "${RG}" --cluster-name "${CLUSTER}" \
  -n nodepool1 \
  --node-image-only \
  --max-surge 1
```

This does not bump Azure Disk CSI and it does not bump the Kubernetes minor. That is the point.

### Move the data plane to Cilium

```bash
az aks update \
  -g "${RG}" -n "${CLUSTER}" \
  --network-dataplane cilium
```

Read this before you run it:

- Every node pool is reimaged, together. There is no per-pool form.
- The IPAM mode does not change. A node-subnet cluster stays node-subnet. Only the data plane swaps.
- If the cluster used Azure NPM or Calico for policy, that engine is removed and Cilium enforces policy only after every node has been reimaged. During the roll, policy behavior is mixed. Do this in a window, on nonprod first, with the policies you actually rely on loaded in a test namespace.
- Windows node pools: unsupported.
- Node auto-provisioning: turn it off, run the update, turn it back on.

IPAM mode to overlay is a different `az aks update`. Do not combine it with the data-plane change in one mental step. They are two reimages if you need both.

## CNI day-to-day

After the data plane is Cilium, policy changes are `NetworkPolicy` or Cilium-compatible policy objects. Speed 1. They do not upgrade the data plane.

You do not pick the Cilium version. AKS ships it. A Fleet cross-cluster network adds a Fleet-managed Cilium mesh (`cilium-agent` and a clustermesh component on the member control plane) and keeps that mesh aligned with AKS releases. Upgrade those member clusters with Fleet update runs so the mesh versions do not diverge. That mesh is not a replacement for the cluster's own Azure CNI data plane. It is the cross-cluster piece.

Pod CIDR and the VNet subnet layout are the slow knobs. Get overlay vs node-subnet right at create.

## CSI

```bash
kubectl get csidriver
kubectl get pods -n kube-system -l app=csi-azuredisk-node
kubectl get storageclass
```

Disk, File, and Blob drivers show up as CSI drivers AKS manages. A cluster upgrade that says it finished and a node plugin that is crash-looping is a real failure. Check it on the canary before Fleet starts stage 2. The test is a PVC on `disk.csi.azure.com`, a pod, a delete, a pod on a different node, and an expansion.

Third-party CSI (a SAN that is not Azure Disk) is Helm, and it is the path when you need a driver fix this week and AKS has not shipped it. Give it a non-default StorageClass so the managed default keeps working.

Azure Disk volumes are zone-scoped when the cluster is zonal. `WaitForFirstConsumer` plus a topology-aware StorageClass avoids the "volume in zone 1, pod in zone 2" pending state.

## At scale: Fleet Manager

Member clusters join a fleet and carry a group name. An update run walks stages. Groups inside a stage run in parallel. The next stage starts after the wait, and after an approval if you set one.

`stages.json`:

```json
{
  "stages": [
    {
      "name": "nonprod",
      "maxAllowedFailures": "0",
      "groups": [
        { "name": "test" },
        { "name": "staging" }
      ],
      "afterStageWaitInSeconds": 86400
    },
    {
      "name": "prod",
      "maxAllowedFailures": "0",
      "groups": [
        { "name": "prod-a" },
        { "name": "prod-b" }
      ]
    }
  ]
}
```

Kubernetes minor plus the node image:

```bash
az fleet updaterun create \
  --resource-group "${RG}" \
  --fleet-name "${FLEET}" \
  --name k8s-1-32-3 \
  --upgrade-type Full \
  --kubernetes-version 1.32.3 \
  --node-image-selection Latest \
  --stages @stages.json
```

Node image only, next week:

```bash
az fleet updaterun create \
  --resource-group "${RG}" \
  --fleet-name "${FLEET}" \
  --name nodeimage-2026-11-21 \
  --upgrade-type NodeImageOnly \
  --node-image-selection Latest \
  --stages @stages.json
```

`one-by-one` is the alternative when you do not want to maintain groups. It upgrades every member sequentially and ignores the careful stage file. Fine for a handful of clusters. Wrong for fifty.

`maxAllowedFailures` of `0` stops the stage on the first failed member. A percentage or a count lets the stage "complete" with failures in it. After every run, read `FailureCount` and the per-member message. A completed run with a non-zero budget is not a clean run.

Update runs honor the planned maintenance window on each member. A member outside its window waits. A member that has fallen off the supported Kubernetes version can be upgraded by AKS outside the run and outside the window, once the grace period ends. Staying current is how you keep that from happening to prod.

Auto-upgrade profiles are the same run, on a schedule, for fleets that have decided the canary stage is enough human attention.

Save the stage file. The next minor should be the same groups. A one-off stage that puts prod in stage 1 is how a test run touches prod.

## Air gap

`az aks create` against Azure public cloud always has a control plane in the region. Private AKS clusters change who can reach the API server. They do not give you a registry-mirror install, and Fleet Manager is an Azure control plane too.

AKS on Azure Local (and the older Azure Stack HCI shape) is the on-prem product. Disconnected operation means the node images, the AKS arc agents, and the Azure Local update payload were staged at the site before you needed them. There is no `az aks upgrade` that reaches out and improvises. Treat Azure Local as its own platform page when you actually have a site: the CLI flags differ, the image import differs, and a public-cloud Fleet run will not upgrade those clusters.

## Traps

- `--node-image-only` when you meant to move Kubernetes, or a `Full` Fleet run when you meant to patch the OS. The upgrade type is the whole content of the change.
- A Cilium data-plane update during a work day. Every pool reimages, and network policy is inconsistent until the last node finishes.
- Combining the IPAM migration and the data-plane migration in one change. When packet flow breaks, you will not know which one did it.
- `--max-surge` with the subnet out of IPs. The new node stays unready. Lower the surge or widen the subnet before you retry. Do not start a second upgrade on top.
- `maxAllowedFailures` set to a percentage on the first prod run. The run completes, two clusters are broken, and the summary looked successful.
- Assuming Azure Disk CSI has an EKS-style `update-addon` you can run this afternoon. It does not. Helm a different driver, or wait for the AKS version that contains the fix.
- Kubenet left in place on a cluster you intended to put on Cilium. The direct data-plane update is rejected. Overlay first.
