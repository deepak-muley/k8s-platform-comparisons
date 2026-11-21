# Upgrades at scale

One cluster is a sequence. Fifty clusters is a policy: who goes first, how many at once, what pauses the rest, and which layer moves (OS image only, Kubernetes minor, CNI/CSI, platform apps).

Pick the layer before you pick the tool. A Kubernetes minor rolls nodes. A node-image CVE does not have to move the minor. A platform-app bump should not roll nodes at all. Running all three in one change window makes the blast radius impossible to read when something fails.

## What "at scale" means on each product

| Product | Built-in fan-out | You still have to build |
| --- | --- | --- |
| OpenShift + ACM | `Placement` selects clusters. A `Policy` or the MultiClusterHub lifecycle can drive `ClusterVersion.spec.desiredUpdate`. Hosted control planes (HyperShift) make the control-plane upgrade cheap because the workers are the only large roll. | The canary `MachineConfigPool` inside each cluster. ACM does not invent that for you. |
| NKP + Kommander | A workspace is the blast radius for platform apps (`nkp upgrade workspace`). Each `NKPCluster` is the blast radius for Kubernetes. Current ClusterClass clusters can be upgraded per cluster from the UI or one CLI command. The documented order upgrades platform apps before Kubernetes. | The order across workspaces. Nothing in Kommander stops you from upgrading every prod cluster the same afternoon unless you split workspaces or you upgrade them one CLI call at a time. |
| GKE | A release channel is a slow fan-out run by Google. Maintenance windows and exclusions are how you shape it. Fleets group clusters for policy and config. Concurrent node-pool upgrades exist so one cluster's pools do not wait in a long line. | A decision about channel membership. Once a cluster is on Regular, Google picks the patch timing inside the window. |
| EKS | `update-cluster-version` and `update-addon` are one cluster. Organizations wrap them in Terraform, Crossplane, or a Step Functions loop. Upgrade insights are the pre-check. | The loop, the canary account, and the reminder that add-ons do not follow the control plane. |
| EKS Anywhere | The management cluster stores every workload `Cluster`. GitOps or Terraform applying those objects is the fan-out. `maxSurge` and `maxUnavailable` are the in-cluster throttle. | A pipeline that applies one cluster, waits for `Ready`, then the next. The management cluster will do whatever you apply. |
| AKS + Fleet Manager | This is the product that matches the question. An update run has stages. Groups inside a stage run in parallel. The next stage waits. `NodeImageOnly` or `Full`. Auto-upgrade profiles repeat it. Member maintenance windows are honored. | Cluster labels that map to groups. A stage JSON. Someone to look at `FailureCount` after a run that "completed" with a non-zero failure budget. |
| Rancher | Downstream cluster upgrades from the Rancher UI or the `clusters.management.cattle.io` object, with a concurrency you set. Fleet (`GitRepo`) rolls manifests and Helm across clusters by target. | A target that is only the canary cluster. RKE2 server upgrades are still one node at a time inside the cluster. |

## A rollout shape that fits all of them

Use the same shape even when the product does not enforce it.

1. **Dev stage.** One cluster. Move the layer you intend to move and nothing else. Watch DaemonSet restarts for CSI/CNI and node replacements for Kubernetes.
2. **Wait.** A real wait (hours, or a business day), not "the command returned".
3. **Prod, low concurrency.** A few clusters at a time, or one node pool at a time inside a large cluster.
4. **Stop condition.** Any failed cluster operator, any `Machine` stuck `Provisioning`, any add-on `DEGRADED`, any node `NotReady` past the drain timeout. The rest of the fleet does not start.

Map that onto the product:

| Stage idea | OpenShift | NKP | GKE | EKS | EKS Anywhere | AKS | Rancher |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Canary | A cluster whose `ClusterVersion` you patch by hand. Inside it, pause `mcp/worker`, unpause a one-node pool. | One cluster in a non-prod workspace. | A cluster on the Rapid channel, or one node pool with `--max-surge-upgrade=1 --max-unavailable-upgrade=0`. | One cluster, one node group, `maxUnavailable=1`. | One workload cluster, `maxSurge: 1`, `maxUnavailable: 0`. | An update stage whose only group is the test group. | A Fleet target selector on the canary label. |
| Wait | Watch `oc get clusterversion` and `oc get mcp` until `UPDATING=False`. | Watch `kubectl get machines` and Kommander `HelmRelease` conditions. | `gcloud container operations list` until the operation is done, then a soak. | `aws eks describe-update` until `Successful`, then the add-on update. | `kubectl get clusters.anywhere.eks.amazonaws.com` and the CAPI machines. | `afterStageWaitInSeconds`, plus a manual approval if you set one. | Wait for the cluster object to go `Active` and all nodes `Ready`. |
| Prod fan-out | ACM placement that excludes clusters still on the canary policy. | `nkp upgrade cluster` in a loop over `kubectl get nkpclusters -A`, filtered by namespace. | Move channel from Rapid to Regular only after the canary soaks, or raise node-pool concurrency on clusters already in the channel. | Your loop, with a concurrency of 5–10 control planes. Node rolls dominate the clock, not the API call. | A pipeline applying cluster YAML in batches. | The prod stage, groups in parallel, `maxAllowedFailures: 0` until you trust it. | Rancher concurrency plus a second Fleet target. |

## Inside one large cluster

Fleet tools do not make a 200-node pool safe. The in-cluster throttle does.

| Platform | Knob | Conservative production value |
| --- | --- | --- |
| OpenShift | `MachineConfigPool.spec.maxUnavailable` | `1` or a small percentage. Pause the pool to freeze a roll. |
| NKP | `topology.cluster.x-k8s.io/upgrade-concurrency` on the cluster | `1` until the first pool succeeds. The default is already 1. |
| GKE Standard | Node pool upgrade strategy. Surge: `maxSurge` / `maxUnavailable`. Or blue-green, which brings up the new set before draining the old. | Surge `maxSurge=1`, `maxUnavailable=0` when capacity is tight. Blue-green when you can pay for a second copy of the pool briefly. |
| GKE Autopilot | Surge, Google-managed. You get maintenance windows and exclusions. | A window that matches when the workload can take a pod move. |
| EKS managed node group | Update config `maxUnavailable` / `maxUnavailablePercentage` | One node, or 10%, whichever is smaller relative to the PDB. |
| EKS Anywhere | `upgradeRolloutStrategy.rollingUpdate.maxSurge` and `maxUnavailable` | `maxSurge: 1`, `maxUnavailable: 0`. |
| AKS | `az aks nodepool update --max-surge` | `1` or `10%`. Surge needs spare IP addresses and spare compute quota. A surge that cannot schedule looks like a hung upgrade. |
| RKE2 | How many agents you restart, and the drain timeout in Rancher | One server at a time, always. Agents in batches that respect PDBs. |

PodDisruptionBudgets are the other throttle, and they will stall a roll on purpose. Before a fleet-wide Kubernetes upgrade, find budgets of `minAvailable: 100%` or `maxUnavailable: 0` on a workload that has one replica. Those block the node drain forever. Fix the budget or the replica count in the canary stage, not during the prod stage.

## Layered cadence

Run these on different clocks.

| Layer | Reasonable cadence | Why it is not tied to the Kubernetes minor |
| --- | --- | --- |
| Node OS CVE | Weekly or as the vendor image publishes | GKE, AKS (`--node-image-only`), and EKS (new AMI, same Kubernetes version) can do this without a minor bump. OpenShift z-streams bundle OS and Kubernetes patch together. |
| Managed CNI / CSI add-on (EKS) | When the vendor publishes a build you need, often between minors | `update-addon` does not require `update-cluster-version`. |
| Third-party CSI | When the storage vendor says so | Helm. Test a volume create, a resize, a snapshot, and a pod mount on an old node and a new node. |
| Platform apps (Kommander, OpenShift operators, Rancher charts) | Before the Kubernetes minor, in the same change window or the one before | They are the layer that breaks on removed APIs. |
| Kubernetes minor | A few times a year, one minor at a time | NKP, EKS Anywhere, RKE2, and OpenShift EUS-to-EUS all punish skipped minors. GKE and AKS channels will move you if you wait past the support window. |

## OpenShift with ACM, concrete

Canary inside the cluster, before you let ACM touch the workers:

```bash
oc patch mcp/worker --type merge -p '{"spec":{"paused":true}}'
oc adm upgrade --to=4.19.6
# masters roll. workers stay.
# put one node in a dedicated MachineConfigPool, confirm the pool completes,
# then:
oc patch mcp/worker --type merge -p '{"spec":{"paused":false}}'
oc get mcp -w
```

At fleet scale, ACM placement selects clusters by label (`env=nonprod` first). The policy sets `spec.desiredUpdate` on `clusterversion/version`. Keep prod in a placement that does not match until the nonprod `ClusterVersion` condition `Available` is true and `oc get co` is clean.

Hosted control planes change the cost. The control plane is pods on a management cluster. Upgrading it does not reboot the workers by itself. The workers still roll when their node pool version moves. Scale the management cluster for the number of hosted control planes, not the number of worker nodes.

## NKP, concrete

```bash
# management cluster kubeconfig
nkp get workspaces
nkp upgrade workspace nonprod-workspace
# platform apps only. then one cluster:
nkp upgrade cluster nutanix \
  --cluster-name nonprod-01 \
  -n nonprod-workspace-ns \
  --vm-image "${VM_IMAGE_NAME}"

kubectl get machines -n nonprod-workspace-ns -w
```

When that cluster's machines are `Running` on the new version, repeat for the prod namespace. Raise pool concurrency only after one pool has finished:

```bash
kubectl annotate cluster "${CLUSTER_NAME}" -n "${NS}" \
  topology.cluster.x-k8s.io/upgrade-concurrency=2
```

On NKP 2.19 and later, ClusterClass workload clusters can take a single upgrade command or a UI action that runs platform apps before Kubernetes and add-ons. The management cluster still has to be on 2.19 first. Non-ClusterClass clusters keep `nkp upgrade platform-apps` as the separate platform-app step.

## GKE, concrete

```bash
gcloud container clusters upgrade "${CLUSTER}" \
  --master \
  --cluster-version "${VERSION}" \
  --location "${LOCATION}"

gcloud container node-pools update "${POOL}" \
  --cluster "${CLUSTER}" \
  --location "${LOCATION}" \
  --max-surge-upgrade=1 \
  --max-unavailable-upgrade=0

gcloud container clusters upgrade "${CLUSTER}" \
  --node-pool "${POOL}" \
  --location "${LOCATION}"
```

For many clusters, put nonprod on Rapid and prod on Regular. The Regular channel receives a minor only after it has soaked on Rapid. Maintenance exclusions freeze a cluster across a peak. They are the supported way to not upgrade during a quarter-close. They are not a way to sit outside the supported version window forever. When the exclusion ends, the channel catches up.

## EKS, concrete

```bash
aws eks update-cluster-version \
  --name "${CLUSTER}" \
  --kubernetes-version 1.32

aws eks wait cluster-active --name "${CLUSTER}"

# add-ons do not move with the command above
aws eks update-addon \
  --cluster-name "${CLUSTER}" \
  --addon-name vpc-cni \
  --addon-version "${VPC_CNI_VERSION}" \
  --resolve-conflicts PRESERVE

aws eks update-addon \
  --cluster-name "${CLUSTER}" \
  --addon-name aws-ebs-csi-driver \
  --addon-version "${EBS_CSI_VERSION}" \
  --resolve-conflicts PRESERVE

aws eks update-nodegroup-version \
  --cluster-name "${CLUSTER}" \
  --nodegroup-name "${NODEGROUP}"
```

A scale loop should do control planes with higher concurrency than node groups. Control planes are an API update. Node groups take a node drain each. Add-ons are cheap and easy to forget; put them in the same pipeline stage as the control plane, gated on `cluster-active`, or the next minor's add-on compatibility check fails months later.

## EKS Anywhere, concrete

Edit `kubernetesVersion` (and `eksaVersion` when the EKS Anywhere minor moves) in the workload cluster YAML. Apply through the management cluster:

```bash
eksctl anywhere upgrade plan cluster -f workload.yaml
eksctl anywhere upgrade cluster -f workload.yaml
```

Or apply the object with `kubectl` / GitOps / Terraform against the management cluster. Keep the management cluster's EKS Anywhere version at most one minor ahead of the workload clusters. A pipeline that bumps every workload cluster's minor in one commit is how you skip the canary.

## AKS Fleet Manager, concrete

```json
{
  "stages": [
    {
      "name": "nonprod",
      "groups": [{ "name": "test" }, { "name": "staging" }],
      "afterStageWaitInSeconds": 86400
    },
    {
      "name": "prod",
      "maxAllowedFailures": "0",
      "groups": [{ "name": "prod-a" }, { "name": "prod-b" }]
    }
  ]
}
```

```bash
az fleet updaterun create \
  --resource-group "${RG}" \
  --fleet-name "${FLEET}" \
  --name "k8s-1-32" \
  --upgrade-type Full \
  --kubernetes-version 1.32.0 \
  --node-image-selection Latest \
  --stages @stages.json
```

Use `--upgrade-type NodeImageOnly` on the weeks you are only patching the node image. A `Full` run on those weeks rolls Kubernetes as well. Groups in a stage run in parallel. Stages run in order. With `maxAllowedFailures` unset or `0`, one failed member stops that part of the run. AKS can still upgrade a cluster that has fallen off the supported version outside your run, after the grace period. Staying inside the window is how you keep Fleet as the only thing that moves prod.

## What to record per cluster

A fleet you cannot describe will eventually run two CNI versions and three CSI versions with no one noticing. Keep this on the cluster object as labels or in the inventory you already have:

- Kubernetes version, and the node pool versions if they differ.
- CNI name and version.
- CSI driver name and version, and the default StorageClass.
- Whether the next air-gap bundle for the *following* version is already in the registry.
- The last successful upgrade date and the change window that owns the cluster.

EKS `aws eks describe-addon --addon-name vpc-cni`, OpenShift `oc get clusterversion`, NKP `kubectl get nkpclusters -A`, and `kubectl get ds -A` for the CNI and CSI DaemonSets are the raw sources. The inventory is only useful if a pipeline writes it after every run. The design that makes this grid the home screen, with each version on its own wave, is [Ideal platform](ideal-platform.md).
