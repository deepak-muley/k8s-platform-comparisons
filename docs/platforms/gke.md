# Google Kubernetes Engine

GKE is a managed control plane. You do not install etcd, and you do not upgrade the VPC CNI by pushing a DaemonSet. You pick a mode, a release channel, and node pools. Google upgrades the control plane on the channel's schedule. You decide how nodes follow, unless you chose Autopilot, in which case Google upgrades those too.

There is no air-gap install of public GKE. Private clusters still talk to Google APIs. The disconnected product is Google Distributed Cloud air-gapped, covered at the end.

GKE is not Cluster API. A node-pool upgrade still replaces the GCE VM, which is the part people compare with CAPX. The difference is the version string. `1.32.8-gke.1056000` is one artifact: you cannot put a new kubelet on the old image, and you can ship a newer `-gke.N` image without moving the Kubernetes minor. Both are new VMs. NKP on CAPX has the first restriction and does not have a separate OS-image command. See [Cluster API](../capi.md).

## Core vs add-on

| | What | Who upgrades it | Speed |
| --- | --- | --- | --- |
| Core | Control plane (API server, etcd, scheduler, controllers) | Google, inside the release channel, or you with `gcloud container clusters upgrade --master` | An API operation. Minutes. No node reboot by itself. |
| Core | Node image and kubelet | Node pool upgrade. Auto-upgrade is on by default for Standard pools in a channel, and always on for Autopilot. | 3. Surge or blue-green. |
| Core | Dataplane V2 (Cilium-based) or the kube-proxy data plane | Google, as part of the node image and the control plane. You enable the feature. You do not pick the Cilium version. | 3 when nodes roll. |
| Managed add-on, essentially core | Compute Engine persistent disk CSI. On by default. | Google, with the GKE version. | 2, inside a GKE upgrade. You cannot pin an upstream CSI tag. |
| Managed add-on | Filestore CSI, Cloud Storage FUSE CSI, HTTP load balancing, NodeLocal DNSCache, backup, managed service mesh | You enable or disable. Google runs the version. | 1 to turn on. The pods roll as speed 2. |
| Add-on | Any CSI or CNI you installed yourself (NetApp, Portworx, a second policy engine) | You, with Helm. GKE will not revert your Helm release. It also will not upgrade it. | 2 |

Autopilot pre-configures the managed add-ons and the surge behavior. You give up node SSH, custom DaemonSets that need host access beyond the allowed set, and the ability to run an arbitrary CNI. Standard is the mode where node pools, max surge, and third-party CSI are normal.

## Release channels

A release channel is the version stream the cluster is subscribed to. Google picks the Kubernetes minor and the `-gke.N` patch from that stream, then auto-upgrades the control plane and the nodes onto the channel's current target. You still choose when, with a maintenance window or an exclusion. You do not pick a Cilium version or a CSI version. Those ride the same GKE version.

A new cluster with neither `--release-channel` nor `--cluster-version` lands on `regular` and that channel's default patch. If you pass a version and no channel, GKE enrolls the cluster in the most mature channel that already contains that version, checking `stable`, then `regular`, then `rapid`. Autopilot can only be in a channel. `None` is not a legal Autopilot value.

See what a channel will actually install in a location before you create:

```bash
gcloud container get-server-config --location "${REGION}" \
  --format='yaml(channels)'
```

Each entry has `channel`, `defaultVersion`, and `validVersions`. Those lists move. Do not hard-code a version from this page.

| Value | What it is | When a new minor shows up | When it becomes the auto-upgrade target | Use it for |
| --- | --- | --- | --- | --- |
| `rapid` | Newest GA minor. Not covered by the GKE SLA. Can ship a version that has no workaround yet. Alpha clusters on Rapid can also see alpha patches, up to about four months early. | 1–2 weeks after upstream Kubernetes GA | 1–2 months after it appears on Rapid | A pre-prod cluster whose job is to break before prod does. |
| `regular` | Default. GA quality, new features once they have soaked on Rapid. | About 2 months after Rapid | About 3 months after it appears on Regular | Most production clusters. |
| `stable` | Same minor, later. Last stop in the soak. | 3–4 months after Regular | About 2 months after it appears on Stable | Production that wants the minor only after Regular has run it. |
| `extended` | Long-term support for one minor. Patch cadence matches Regular during the 14-month standard support window, then security patches for about 10 more months. 24 months from when that minor opened on Regular. Around 2 months before the end, GKE starts the next minor unless an exclusion is still legal. | Same minors as Regular | Same patch targets as Regular. It does not jump minors during the extended window the way Regular does. | A cluster that cannot take a minor on the Regular calendar. Not Autopilot. Not Windows node pools, not alpha, not an explicit Kubernetes beta API. Extended-support pricing starts when the minor enters the extended window. |
| `None` | Deprecated. Removed 14 June 2027, after which remaining clusters are put on `stable`. New customers cannot set it. Control planes still auto-upgrade. Minor auto-upgrade follows the Stable date. Available minors and the default version follow Regular. Patch versions available follow Rapid. | — | Stable's minor date | Not for a new cluster. A maintenance exclusion is how you pause a channel. |

Critical patches can land on every channel, including a version that channel has not promoted yet. A maintenance exclusion delays an ordinary auto-upgrade. It does not let the cluster sit on a minor past end of support. A `None` cluster only gets the "no upgrades" exclusion (90 days), not the "no minor upgrades until end of support" exclusion a channel cluster gets.

`--patch-update=accelerated` makes patch auto-upgrades start when the patch is available, before Google marks it the channel target. It does not pull the next minor early. The default is `default`.

## Create

Standard, regional, Dataplane V2, Regular channel:

```bash
gcloud container clusters create "${CLUSTER}" \
  --location "${REGION}" \
  --release-channel regular \
  --enable-dataplane-v2 \
  --enable-ip-alias \
  --machine-type n2-standard-8 \
  --num-nodes 3 \
  --workload-pool "${PROJECT}.svc.id.goog"

gcloud container clusters get-credentials "${CLUSTER}" --location "${REGION}"
kubectl get nodes
kubectl get storageclass
kubectl get ds -n kube-system
```

What that command sets, against what you get if you leave the flag off:

| Flag | Value in the command | If you omit it | What it does | After create |
| --- | --- | --- | --- | --- |
| `--location` | A region, such as `us-central1` | gcloud's `compute/region` or `compute/zone`, if one is configured. Otherwise the command fails. | A region builds a regional control plane, replicated across zones. A zone (`us-central1-a`) builds a zonal control plane. `--region` and `--zone` are the older spellings of the same choice. | The control plane location does not move. Build a new cluster. |
| `--release-channel` | `regular` | `regular`, at that channel's `defaultVersion`. | Subscribes the cluster to one version stream. | You can change channel. Moving to a channel that does not contain the running minor fails until you upgrade onto a minor that channel has. |
| `--enable-dataplane-v2` | set | Off on Standard. On for every new Autopilot cluster, with no flag. | Replaces kube-proxy with GKE's eBPF dataplane (Cilium-based). NetworkPolicy is built in. Do not also pass `--enable-network-policy`. GKE rejects that combination (`DatapathProvider=ADVANCED_DATAPATH`). You do not pick the Cilium version. | Create-time only. You cannot turn it on or off later. A cluster that needs it and does not have it is a new cluster. |
| `--enable-ip-alias` | set | Same result. Omitting both this and `--no-enable-ip-alias` already creates a VPC-native cluster. | Pod and Service IPs come from alias ranges on the subnet, not from Google Cloud Routes. `--no-enable-ip-alias` is the routes-based cluster. Do not use it. Pod and Service range sizes are create-time. | The mode does not change. Changing the ranges is a new cluster. |
| `--machine-type` | `n2-standard-8` | `e2-medium` | Machine type of the default node pool only. | Immutable on the pool. Add `pool-b` and drain the old pool. |
| `--num-nodes` | `3` | `3` | Nodes **per zone** in the default pool, not the cluster total. A regional location uses three zones, so this command creates 9 VMs. `--num-nodes=0` creates no default pool; you add pools yourself, and you cannot combine that with `--machine-type`. | Resize the pool, or add another pool. |
| `--workload-pool` | `PROJECT_ID.svc.id.goog` | Off, unless the server default for that gcloud has changed. Autopilot turns it on. The only legal value is the project id plus `.svc.id.goog`. | Workload Identity. A Kubernetes service account can act as an IAM service account. Pods stop using the node service account. | Can be enabled later. New node pools then use the GKE metadata server. Confirm with `gcloud container clusters describe`. |

Autopilot ignores node flags. You do not pass `--machine-type`, `--num-nodes`, or `--enable-dataplane-v2`:

```bash
gcloud container clusters create-auto "${CLUSTER}" \
  --location "${REGION}" \
  --release-channel regular
```

Autopilot is Dataplane V2, Workload Identity, surge upgrades, and Google-managed nodes. There is no SSH to the node and no arbitrary CNI.

### Flags this command does not pass

These are on or off by default. Set them only when the default is wrong. `--addons` on create is a set, not a patch: the add-ons you name are enabled and the others are disabled. Use `--update-addons` later when you mean to change one.

| Flag | Default when omitted | What it means |
| --- | --- | --- |
| `--cluster-version` | The channel's `defaultVersion` | Pins the first version. The channel still auto-upgrades off that pin onto the next target unless an exclusion says otherwise. A version from a newer channel is legal only for a patch of a minor your channel already has. |
| `--enable-autoupgrade` | On | Node pools follow the control plane. `--no-enable-autoupgrade` leaves a pool behind until it falls out of the skew window and GKE moves it anyway. Prefer a maintenance exclusion. |
| `--enable-autorepair` | On for COS and Ubuntu images | GKE recreates a node that fails its health checks. `--no-enable-autorepair` turns that off for the default pool. |
| `--max-surge-upgrade` / `--max-unavailable-upgrade` | Surge 1. Pass both flags together when you set either. | Extra VMs created during a pool upgrade, and how many existing nodes may be down at once. Surge 1 and unavailable 0 keeps the old node until the new one is Ready, and needs a spare IP and quota. |
| `--disk-size` | 100 GB | Boot disk of each node in the default pool. Not the size of persistent volumes. |
| `--disk-type` | `pd-balanced` on 1.24 and later. `pd-standard` before that. | Boot disk type. Also `pd-ssd`, `hyperdisk-balanced`, `hyperdisk-extreme`, `hyperdisk-throughput`. |
| `--image-type` | Server default, which is Container-Optimized OS with containerd. List names with `gcloud container get-server-config`. | Node OS. `UBUNTU_CONTAINERD` is the other common value. The pool keeps that type across upgrades. |
| `--addons` | `HttpLoadBalancing` and `HorizontalPodAutoscaling` enabled. | Other names you opt into: `GcePersistentDiskCsiDriver`, `GcpFilestoreCsiDriver`, `GcsFuseCsiDriver`, `NodeLocalDNS`, `BackupRestore`, `ConfigConnector`, `LustreCsiDriver`. GCE PD CSI is already the storage path on current clusters. Do not pass `--addons` just to turn one extra driver on, or you will turn the defaults off. |
| `--enable-managed-prometheus` | On for 1.27 and later | Google's managed Prometheus collector. `--no-enable-managed-prometheus` turns it off. |
| `--enable-shielded-nodes` | On since 1.18 | Shielded node credential bootstrapping. |
| `--enable-network-policy` | Off | Policy enforcement on the kube-proxy dataplane. Illegal once `--enable-dataplane-v2` is set, because policy is already in that dataplane. |
| `--enable-private-nodes` | Off | Nodes get internal IPs only. Needs a VPC-native cluster. Pair with `--enable-private-endpoint` when the API server should not have a public endpoint either. A private cluster still reaches Google APIs. It is not an air gap. |
| `--enable-autoscaling` | Off. Pair with `--min-nodes` and `--max-nodes`. | Cluster autoscaler on the default pool. Autopilot scales without this flag. |
| `--enable-kubernetes-alpha` | Off | Every alpha API, no upgrades, cluster deleted after 30 days, no SLA. Not a way to test a channel. |

## Day-2 config

Most node-pool fields are immutable. The pattern is a new pool, a cordon of the old pool, a drain, a delete.

```bash
gcloud container node-pools create pool-b \
  --cluster "${CLUSTER}" \
  --location "${REGION}" \
  --machine-type n2-standard-16 \
  --num-nodes 3 \
  --max-surge-upgrade 1 \
  --max-unavailable-upgrade 0

kubectl cordon -l cloud.google.com/gke-nodepool=default-pool
kubectl drain -l cloud.google.com/gke-nodepool=default-pool \
  --ignore-daemonsets --delete-emptydir-data

gcloud container node-pools delete default-pool \
  --cluster "${CLUSTER}" \
  --location "${REGION}"
```

Things you can change in place on the cluster object: release channel, maintenance window, maintenance exclusion, add-on flags, some network policy switches. Example, a window and an exclusion:

```bash
gcloud container clusters update "${CLUSTER}" \
  --location "${REGION}" \
  --maintenance-window-start 2026-01-01T09:00:00Z \
  --maintenance-window-end 2026-01-01T17:00:00Z \
  --maintenance-window-recurrence 'FREQ=WEEKLY;BYDAY=TU,WE,TH'
```

A maintenance exclusion is how you freeze a cluster across a known peak. When it ends, the channel catches the cluster up. It is not a permanent opt-out.

Enable a managed CSI add-on (Filestore as the example; the flag name follows the add-on):

```bash
gcloud container clusters update "${CLUSTER}" \
  --location "${REGION}" \
  --update-addons GcpFilestoreCsiDriver=ENABLED
```

That rolls the Filestore CSI pods. It does not reimage nodes. Confirm the exact flag with `gcloud container clusters update --help` on the gcloud you have. Add-on flag names have moved.

A StorageClass is still just a StorageClass. `standard-rwo` / `premium-rwo` are the GCE PD classes GKE creates. Adding your own does not touch the driver version.

## Upgrade

List what this location will actually accept:

```bash
gcloud container get-server-config --location "${REGION}"
```

Control plane, then the pool:

```bash
gcloud container clusters upgrade "${CLUSTER}" \
  --master \
  --cluster-version "${VERSION}" \
  --location "${REGION}"

gcloud container node-pools update "${POOL}" \
  --cluster "${CLUSTER}" \
  --location "${REGION}" \
  --max-surge-upgrade 1 \
  --max-unavailable-upgrade 0

gcloud container clusters upgrade "${CLUSTER}" \
  --node-pool "${POOL}" \
  --cluster-version "${VERSION}" \
  --location "${REGION}"
```

The control plane and the nodes are different operations and can differ by the supported skew (nodes older than the control plane, inside the policy). Auto-upgrade exists so a pool you forgot does not sit there past that skew. Prefer leaving auto-upgrade on and controlling *when* with the maintenance window, rather than turning it off.

Surge with `maxUnavailable=0` keeps the old node until the new one is Ready. You need a spare IP and spare quota for the surge node. Blue-green brings up a second copy of the pool and then cuts over. Use it when a surge of 1 is too slow and you can pay for the overlap. Autopilot pools use surge. You do not pick blue-green there.

Watch:

```bash
gcloud container operations list --location "${REGION}"
kubectl get nodes -L cloud.google.com/gke-nodepool
kubectl get pods -n kube-system -o wide
```

GCE PD CSI and Dataplane V2 move as part of this. There is no second command, and there is no way to stay on the old GCE PD CSI after the node has rolled to an image that contains the new one.

### Node image only

When a GKE patch is a node image rebuild on the same Kubernetes minor, upgrading the node pool without a `--cluster-version` minor bump is the CVE path. The pool's version string gains a new `-gke.N` build. Workloads move. The Kubernetes minor stays. Use that for the weekly image cadence, and use a minor bump on the channel cadence.

## CNI

What you can change:

- `NetworkPolicy` objects. Speed 1. On Dataplane V2 they program the eBPF dataplane. On a cluster without it they need `--enable-network-policy`, which is a different dataplane and not something you add beside Dataplane V2.
- Dataplane V2 observability, where the release still exposes it (`--enable-dataplane-v2-metrics`, `--enable-dataplane-v2-flow-observability`). Those do not change the CNI.

What you cannot change:

- Dataplane V2 itself. `--enable-dataplane-v2` is create-time. An existing Standard cluster stays on kube-proxy.
- The Cilium version inside Dataplane V2.
- Pod CIDR, after creation.
- A plan to "helm upgrade cilium" on a GKE cluster. The DaemonSet you see is not yours. An upgrade will fight you, and a node image will put Google's manifest back.

If you need a CNI feature Dataplane V2 does not expose, the honest options are a GKE feature request, or a cluster on a platform where you own the CNI (NKP, EKS Anywhere, RKE2). Running a second overlay on top of GKE is how clusters get two places that drop packets.

## CSI

GCE PD:

```bash
kubectl get csidriver
kubectl get pods -n kube-system -l app=gcp-compute-persistent-disk-csi-driver
kubectl get storageclass
```

You enable features (snapshot class, hyperdisk, volume attributes) through the GKE API and StorageClass parameters. You do not upgrade the driver image. A third-party CSI is a normal Helm release in a namespace you pick, with a StorageClass whose `provisioner` is that driver. Keep GCE PD as the default unless you mean to change the default. Two defaults are a coin flip.

Test after any GKE node upgrade: a new PVC, a pod that mounts an existing regional PD, a resize. Detach bugs show up when a pod moves onto a node that just joined with the new CSI node plugin.

## At scale

Put the canary cluster on Rapid and production on Regular. Regular does not receive a minor that Rapid has not already run. That is the fleet mechanism. You do not write a loop unless you are doing something the channel will not do (a one-off image, a config push).

For a config push across a fleet, use Terraform against the GKE API, or Config Sync / Config Connector if the change is a Kubernetes object. Fleet membership groups clusters for policy. It does not replace the release channel as the upgrade engine.

Concurrent node-pool upgrades (where your GKE version exposes them) shorten a cluster that has many pools. They do not remove the need for PDBs that can actually spare a pod.

Maintenance exclusions are the freeze switch. Record when they end. A cluster that comes out of a long exclusion and jumps two patch builds in one window surprises people who thought "excluded" meant "unchanged until we say so".

## Google Distributed Cloud air-gapped

This is the air-gap counterpart. You do not mirror `gcr.io` yourself and point GKE at it. The GDC appliance has the versions. Automatic node repair, automatic cluster upgrade, cluster autoscaler, and Autopilot are not part of GKE-on-GDC. You upgrade by hand.

```bash
kubectl --kubeconfig "${MANAGEMENT_API_SERVER}" \
  get userclustermetadata \
  -o custom-columns='TARGET_VERSION:.spec.kubernetesVersion'

kubectl --kubeconfig "${MANAGEMENT_API_SERVER}" apply -f - <<EOF
apiVersion: cluster.gdc.goog/v1
kind: UserClusterUpgradeRequest
metadata:
  name: ${CLUSTER_NAME}-upgrade
  namespace: ${CLUSTER_NAMESPACE}
spec:
  clusterRef:
    name: ${CLUSTER_NAME}
  targetVersion: ${TARGET_VERSION}
EOF
```

`targetVersion` must be one of the versions `UserClusterMetadata` listed. A request for anything else fails in-site. It does not fall back to the public channel. Budget on the order of 20 minutes for the cluster upgrade itself, then verify nodes the way you would on GKE. Dataplane V2 is the CNI direction on current GDC as well.

Because nothing auto-upgrades, a calendar is part of the platform. The public-GKE habit of "the channel will get it" does not exist here.

## Traps

- Disabling node auto-upgrade on Standard and then missing the skew window. The control plane moves anyway. Kubelets that are too old get cordoned by the version policy at the worst time.
- Assuming a private cluster is an air gap. It is a private API endpoint. Image pulls and the upgrade service still depend on Google.
- Helm-upgrading the GCE PD CSI DaemonSet. The next node image puts it back, and the two copies fight over the CSI socket.
- A surge upgrade with no free IPs in the node subnet. The new node never becomes Ready, the old node never drains, the operation runs until it times out.
- Changing machine type "in place". Create `pool-b`.
- Two default StorageClasses after enabling Filestore or a third-party driver.
