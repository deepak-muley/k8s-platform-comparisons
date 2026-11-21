# Versioning

What each digit is, and which digit moves for a Kubernetes patch, an OS rebuild, a CVE, and an add-on. How those digits are cut across the life of one minor, and how a platform team rolls each kind through hundreds of clusters, is [One minor, four clocks](#one-minor-four-clocks-kubernetes-135). What that change does to a node is [CVE scope](cve-scope.md). How long the old digit stays supported is [Compatibility](compatibility.md). The fan-out is [Upgrade at scale](scale.md).

Documented November 2026. The strings below are the shapes those products print. A channel list from the day you upgrade beats a string copied off this page.

## The upstream stem

Every product below hangs its own suffix on [Kubernetes semantic versions](https://kubernetes.io/releases/version-skew-policy/): `MAJOR.MINOR.PATCH`, written `1.36.6`.

| Field | What it is | When it moves |
| --- | --- | --- |
| Major (`1`) | A break in the Kubernetes API. It has stayed at 1. | A future incompatible release. |
| Minor (`36`) | Features and API removals. About every 15 weeks. | The minor upgrade. |
| Patch (`6`) | Bug fixes and CVEs for that minor. The project patches the newest three minors. | A patch release. No features. |

CoreDNS, etcd, and the CNI and CSI projects have their own `MAJOR.MINOR.PATCH`. Upstream Kubernetes does not encode an OS version. The vendor strings below are how each product stuffs the OS, the rebuild, and the add-on back into a version you can point at.

## Which number moves

| | Kubernetes minor | Kubernetes patch, or a CVE in the apiserver or kubelet | OS rebuild, Kubernetes patch unchanged | Vendor CNI or CSI | An add-on you installed |
| --- | --- | --- | --- | --- | --- |
| OpenShift | `4.y` → `4.y+1`. The y is one Kubernetes minor. | The z in `4.y.z`. The payload's Kubernetes patch moves with it. | The same z. RHCOS is `machine-os` inside that payload. | Payload CNI: the same z. An optional operator: its CSV in the `v4.y` index. | That operator's CSV, or your chart version. |
| NKP | The Kubernetes field in the image name, and usually the product minor `2.y`. | The Kubernetes field in the image name (`1.36.2`). | A new image filename whose Kubernetes field is the same, when the release publishes one. | The build bundled in that NKP release. | Your chart version. A catalog app is its own chart. |
| GKE | `1.Y` → `1.Y+1`, still with a `-gke.N`. | A new patch digit, and a new `-gke.N`. | Another `-gke.N` on the same `1.Y.Z`, when `get-server-config` lists one. | The same `1.Y.Z-gke.N`. | Your chart version. |
| EKS | The minor you set, `1.36` → `1.37`. | Control plane: the next `eks.N` for that minor. AWS applies it. Nodes: the patch inside the AMI release version. | The AMI date (`v20261107` → the next date). The minor in the AMI name stays. | `vX.Y.Z-eksbuild.N` on that add-on. | The same add-on form, or your chart version. |
| EKS Anywhere | `kubernetesVersion` (`1.35` → `1.36`) and, on the product cadence, `v0.M` → `v0.M+1`. | A patch of `v0.M.x`. The Kubernetes patch is the one that release bundles. | A new `osImageURL` or template at the same `kubernetesVersion`. | Cilium: the `v0.M.x` release. CSI you installed: your chart. | `spec.packageVersion`. |
| AKS | `1.Y.Z` → `1.Y+1.z` on `kubernetesVersion`. | The patch digit on `kubernetesVersion` and on the pool's `orchestratorVersion`. | `nodeImageVersion` date (`202604.07.0`). A security-only VHD appends `-YYYY.MM.DD`. | The AKS version. No separate pin. | Your chart version. |
| RKE2 | `v1.Y` → `v1.Y+1`, then `+rke2r1` of the first patch you take. | `v1.Y.Z+rke2r1` of the new patch. | The host's package version (`rpm`, `apt`). The RKE2 string stays. | Bundled chart: `+rke2rN`. A chart you installed: its version. | Your chart version. |
| Rancher | The manager minor `2.15`, which is a support matrix, not a Kubernetes minor. | The manager patch `2.15.3`. Downstream Kubernetes stays. | Downstream, on that cluster's row above. | Downstream, on that cluster's row. | The feature chart's version (monitoring, logging, Fleet). |

Support extensions do not add a digit. OpenShift EUS is an even `4.y` on the `eus-4.y` channel. GKE extended support is the same `1.Y.Z-gke.N` on the `extended` channel. EKS extended support is the same minor, with AWS still publishing `eks.N`. AKS LTS is the same `1.Y.Z` on the Premium `AKSLongTermSupport` plan, and only the two newest patches of that minor stay in support. EKS Anywhere extended Kubernetes is the same `kubernetesVersion`, and the patch arrives inside a later `v0.M.x` for clusters that have a license token.

## One minor, four clocks: Kubernetes 1.35

A finding has a layer. The layer decides the number that changes, and the number decides the workflow. Kubernetes `1.35.z` moves only when the CVE or the bug is in Kubernetes itself (the apiserver, the kubelet, their Go build, a vendored dependency of those binaries). An app image, an add-on, a CSI driver image, and the node OS each have their own counter. They can ship on a Tuesday while every cluster stays on `1.35.10`.

The dates below are the ones this repo already records. Rules: [Kubernetes patch releases](https://kubernetes.io/releases/patch-releases/). Calendar: [Compatibility](compatibility.md). `1.35.0` is 17 Dec 2025. Standard patches run until 28 Dec 2026. Maintenance mode runs until the end of life on 28 Feb 2027.

### How upstream builds the next integer

There is one release branch, `release-1.35`. Fixes are cherry-picked onto it. About once a month the patch release managers tag whatever has merged. A cherry-pick that misses the deadline rides the next tag. A critical fix can be tagged early, between monthlies. The early tag is the next integer. The release note says it was out of band. Nothing in the version string says "CVE".

```text
1.35.0          17 Dec 2025   minor goes GA
└─ 1.35.1       10 Feb 2026   monthly. January's cherry-picks shipped inside February's tag.
   └─ 1.35.2    26 Feb 2026   out of band, sixteen days later. A newer Go, for Go CVEs. No other changes.
      └─ 1.35.3 …            each later month. Every cherry-pick since the previous tag: bugs and CVEs in the same integer.
         └─ maintenance      28 Dec 2026 → 28 Feb 2027
            │                a new integer only for a CVE that has an id,
            │                a dependency or base-image fix, or a critical core-component break
            └─ end of life   28 Feb 2027
                             the branch stops. The next Kubernetes CVE is fixed on 1.36 and 1.37.
```

`1.35.1` and `1.35.2` are the pattern to keep. `1.35.1` is a bag of everything that was ready. `1.35.2` is a single purpose, and it is still just `+1`. Two CVEs in the same week share one tag. There is no `1.35.2-cve1`.

Through 28 Dec 2026 that bag is the normal patch: bug fixes and security fixes together. From 28 Dec 2026 to 28 Feb 2027 the bag gets smaller. Release managers may cut a tag for three reasons only: a vulnerability that has a CVE id, a dependency issue (including a base-image update), or a critical break in a core component. A routine bugfix found in January 2027 is fixed on `1.36.z`. Clusters still on 1.35 do not get a backport integer for it.

After 28 Feb 2027 upstream will not cut `1.35.<next>`. A platform team that is still on 1.35 takes the fix as a minor hop to a patched 1.36 or 1.37, or as the vendor suffix below, when that product's extended window is still open. The Kubernetes patch digit stays at the last upstream tag. The vendor digit moves.

| Still on 1.35 after upstream end of life | What keeps moving | What you select |
| --- | --- | --- |
| OpenShift 4.22 (this is the minor that ships Kubernetes 1.35) | `4.22.z`, for as long as that minor's support or EUS term publishes z-streams | The z-stream. `oc adm release info` shows which `1.35.z` it contains. |
| GKE `extended` | `-gke.N`, security fixes | The build `get-server-config` lists for that patch. |
| EKS extended support | `eks.N`, AMI dates, and add-on `eksbuild.N` | The minor stays `1.35`. AWS rolls the platform version. You roll the AMI and the add-ons. |
| AKS LTS | The two newest `1.35.z` patches AKS is still offering | `-k` with one of those two patches. |
| EKS Anywhere | A later `v0.M.x` that still lists `kubernetesVersion: "1.35"`, with a license token once the minor is in extended support | `eksaVersion`. The Kubernetes field stays `1.35`. |
| NKP, RKE2, Rancher | Whatever that product's matrix still lists | The image name, or `v1.35.z+rke2rN`. A Rancher minor that has dropped 1.35 will not grow a new rebuild for it. |

### Four findings on a cluster that is already at 1.35.10

These are siblings. Taking one does not take the others. `1.35.10` is the patch [Compatibility](compatibility.md) records as current for November 2026. Substitute the patch you are actually running. The shape of the tree does not change.

```text
cluster today
kubernetes 1.35.10
│
├─ 1. App container CVE
│     payments:1.8.3  →  payments:1.8.4
│     1.35.10 stays. The CSI driver stays. The node image stays.
│     Same on every platform. Speed 2. The DaemonSet or Deployment rolls.
│
├─ 2. Urgent add-on, other than CSI
│     the add-on's own version. 1.35.10 stays, except where the add-on is inside the platform build.
│     ├─ you installed it (cert-manager, your ingress, a Helm chart)
│     │     chart 1.16.2 → 1.16.3. Speed 2.
│     ├─ EKS managed (vpc-cni, coredns, kube-proxy)
│     │     v1.19.2-eksbuild.1 → v1.19.2-eksbuild.2, or a new upstream at eksbuild.1.
│     │     update-addon. Speed 2. kube-proxy's upstream half is the Kubernetes patch; a packaging CVE is still only eksbuild.
│     ├─ RKE2 bundled (CoreDNS, Canal, the ingress chart RKE2 ships)
│     │     v1.35.10+rke2r1 → v1.35.10+rke2r2. The Kubernetes patch stays. Nodes restart.
│     └─ baked into the platform (OpenShift DNS and ingress, GKE kube-proxy / Dataplane V2, AKS Azure CNI)
│           the next platform build in the vendor table below. You wait for that build.
│
├─ 3. CVE in the CSI container
│     the driver image. Not the kernel, and not a volume that is already attached.
│     ├─ EKS aws-ebs-csi-driver
│     │     v1.40.0-eksbuild.1 → v1.40.0-eksbuild.2. Speed 2. Nodes stay up.
│     ├─ a chart you installed (NKP pre-provisioned, RKE2, EKS Anywhere, any third-party CSI)
│     │     chart 1.40.0 → 1.41.0. Speed 2.
│     └─ GKE GCE PD, AKS Azure Disk / File / Blob, OpenShift payload CSI, NKP's bundled driver
│           the platform build. No CSI-only integer.
│           NKP bundled: the build inside the NKP release, via nkp upgrade addons.
│
└─ 4. CVE in the OS
      kernel, containerd, OS packages. The node rolls. Speed 3.
      ├─ GKE            1.35.10-gke.N  →  1.35.10-gke.M     new VM. Kubernetes patch stays when a same-patch build exists.
      ├─ EKS            AMI 1.35.10-20261107 → 1.35.10-20261121
      │                 new instance. Minor stays 1.35. Add-ons stay.
      ├─ AKS            node image …-202611.07.0 → …-202611.21.0
      │                 or a security VHD suffix -2026.11.21
      │                 orchestratorVersion stays 1.35.10
      ├─ RKE2           the OS package version. v1.35.10+rke2rN stays. Reboot when the kernel changed.
      ├─ EKS Anywhere   new osImageURL or template. kubernetesVersion stays 1.35. New machine.
      ├─ NKP            a new image filename that still contains 1.35.10, when the release publishes one. New VM.
      └─ OpenShift      4.22.z → 4.22.z+1
                        machine-os and the kubelet move together. This row is also a Kubernetes patch.
```

The health check follows the layer. An app wave is the app's readiness. A CSI wave creates a volume and remounts an existing one. An OS wave is a node that comes back `Ready` and rejoins the workload. A Kubernetes patch wave is the apiserver on the new patch, then the kubelet.

### What you type when the finding is a Kubernetes patch

Upstream tagged `1.35.2` for a Go CVE. The platform team does not type `1.35.2` everywhere. They type the vendor's name for "the build that contains `1.35.2`".

| Platform | Select this | Nodes |
| --- | --- | --- |
| OpenShift | The `4.22.z` whose `oc adm release info` prints kubernetes `1.35.2`. | Reboot onto that z-stream's `machine-os`. Pause `mcp/worker` and the control plane still moves first. |
| NKP | The image whose name contains `1.35.2`, from the NKP release that built it. | New VM. |
| GKE | `1.35.2-gke.N` from `get-server-config`. `--master` first, then each pool. | New VM when the pool moves. |
| EKS | Minor stays `1.35`. A new `eks.N` carries patch `1.35.2` on the control plane, and AWS applies it. Nodes: AMI release `1.35.2-YYYYMMDD`. | New instance when the node group moves. Add-ons stay until `update-addon`. |
| EKS Anywhere | The `v0.M.x` that bundles patch `1.35.2`. `kubernetesVersion` stays `1.35`. | New machine. |
| AKS | `-k 1.35.2` on the control plane, then the pool's `orchestratorVersion`. | New VM when the pool moves. `--node-image-only` does not pick up `1.35.2`. |
| RKE2 | `v1.35.2+rke2r1` on servers, then agents. | Same host, service restart. The OS packages stay. |

Read the release note before you pick the ring. `1.35.2` changed Go and nothing else, so the blast radius is the Kubernetes binaries. The CSI chart, the app image, and the OS image stay on their pins. `1.35.1` was a month of unrelated fixes in one integer, so the canary has more to watch, and the command is the same.

### A hundred clusters

One finding, one number, one wave. The other columns stay put, which is how you can tell a bad CSI roll from a bad kernel roll. The stage shape is the one in [Upgrade at scale](scale.md): a canary, a real wait, then production at low concurrency, and a stop when one cluster goes bad.

1. Name the layer from the tree. Write `from` and `to` for that layer only.
2. Read the phase when the layer is Kubernetes. Before 28 Dec 2026, take the next `1.35.z`. From then until 28 Feb 2027, take a `1.35.z` whose note matches the maintenance list (CVE id, dependency or base image, critical core). After 28 Feb 2027, a Kubernetes CVE is a minor hop or a vendor suffix, not another `1.35.z`.
3. Put that exact artifact in the registry or the image library. The first canary cluster should not be the one that discovers the mirror is short.
4. Canary a handful of clusters. Speed 2: watch the DaemonSet or Deployment. Speed 3: watch one drain with `maxSurge` 1 and `maxUnavailable` 0.
5. Soak. Hours when pods rolled. A business day when nodes rolled.
6. Fan-out with the product's mechanism: an AKS Fleet stage, an ACM placement, a GKE channel or a pool loop, one NKP cluster at a time, an EKS loop you own, a Rancher target. A container roll can move tens of clusters at once. A node roll stays at a few clusters, because each cluster is itself a queue of drains.
7. Stop the ring on one degraded cluster. Every cluster not in the ring is still on the old number.

During the maintenance window the fleet is already walking from 1.35 to 1.36. A finding in the middle uses the same tree. It does not get a special digit, and it does not restart the minor hop.

| The finding | Clusters still on 1.35 | Clusters already on 1.36 |
| --- | --- | --- |
| Upstream cuts `1.35.11` (CVE, dependency, or critical) | Roll `1.35.11` with the table above. | Take `1.36`'s patch that contains the same fix. Leave them on 1.36. |
| Upstream cuts nothing, because it is a routine bug | The fix is on `1.36.z`. Finish the hop. | Take that `1.36.z`. |
| App image, or a CSI / add-on chart you own | Roll the chart. `1.35.10` stays. The same chart version lands on 1.36 clusters when the chart supports both minors. | The same wave, same chart version. |
| OS image | The 1.35 image rebuild. A 1.36 image does not patch a 1.35 node. | That minor's own image rebuild. |

A speed 3 wave (OS, or a Kubernetes patch that replaces nodes) and another speed 3 wave stay off the same clusters on the same day. A speed 2 wave (app, EKS add-on, a Helm CSI) can follow on the clusters that have finished draining. Mixing them into one change makes a failed canary unreadable.

## OpenShift

`4.19.6`

| Field | Example | What it is |
| --- | --- | --- |
| Major | `4` | The OpenShift 4 release. Minor releases are the upgrade unit inside it. |
| Minor | `19` | A y-stream, about every 4 months. One Kubernetes minor for the life of that y-stream. Deprecations happen here. |
| Z-stream | `6` | Fixes for that minor. A CVE in the payload, a bug, and the matching RHCOS commit share this number. |

Red Hat's names for those three are major, minor, and z-stream. The release image tag adds the architecture: `quay.io/openshift-release-dev/ocp-release:4.19.6-x86_64`. `ClusterVersion` stores `4.19.6`. The architecture is on the image, and a disconnected upgrade pins the digest.

Through 4.22 each minor has shipped Kubernetes `1.(minor + 13)`:

| OpenShift | Kubernetes minor | RHEL content in RHCOS |
| --- | --- | --- |
| 4.18 | 1.31 | 9.4 |
| 4.19 | 1.32 | 9.6 |
| 4.20 | 1.33 | 9.6 |
| 4.21 | 1.34 | 9.6 |
| 4.22 | 1.35 | 9.8 or 10.2 |

The Kubernetes patch is whichever patch that z-stream's payload contains. `oc adm release info` prints it next to `machine-os`. A 4.22 cluster is built from one of those two RHEL contents; `machine-os` says which. Confirm the minor you are on there. Source for the RHEL column: the [RHCOS FAQ](https://github.com/openshift/os/blob/master/docs/faq.md). Source for the three fields: [OpenShift 4.22 API overview](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html-single/api_overview/index).

The upgrade target is `4.y.z`. RHCOS is the `machine-os` build inside that payload. `416.94.202409040013-0` is OpenShift 4.16, RHEL 9.4 (`94`), then the build timestamp and a revision.

A z-stream that fixes a kernel CVE is a new `4.y.z` and a new `machine-os` build together. Channels (`stable-4.y`, `fast-4.y`, `eus-4.y`, `candidate-4.y`) decide when Cincinnati offers that same z-stream. Errata type is metadata on it: RHSA for security, RHBA for a bug fix, RHEA for an enhancement. A z-stream can carry more than one advisory.

Even minors (4.18, 4.20, 4.22) are EUS. Full support, then maintenance to about 18 months, then optional EUS terms that keep publishing z-streams. The version you upgrade to is still `4.y.z`. The lifecycle is the [OpenShift life cycle](https://access.redhat.com/support/policy/updates/openshift).

Optional operators (logging, GitOps, a CSI subscription) use the CSV version from the catalog index `v4.y`. That version can move while the payload z-stream stays. A payload operator (OVN-Kubernetes, monitoring, the ingress operator) uses the z-stream.

```bash
oc get clusterversion version -o jsonpath='{.status.desired.version}{"\n"}'
oc adm release info
oc get csv -A
```

## NKP

Two versions, on purpose. `nkp version` prints the product. The Prism image name prints the Kubernetes version and the OS.

| | Example | What a bump is |
| --- | --- | --- |
| Product | `v2.19.0`, patch `v2.19.1` | Major `2`, minor `19`, patch `1`. The CLI, Kommander, and Konvoy share it. One product minor at a time. |
| Kubernetes | `1.36.2` in the image name | The kubelet and the control-plane version baked into that image. NKP 2.19 defaults to 1.36.2 and still supports the 1.35.x family. |
| Node image | `nkp-rocky-9.8-release-cis-1.36.2-….qcow2` | OS (`rocky-9.8`), variant (`cis`), Kubernetes (`1.36.2`), then the build. The CLI looks the image up by this name. |

A product patch is `2.19.1`. Read that release's notes for whether the Kubernetes field changed. A new image whose name still contains `1.36.2` is an OS or packaging rebuild at the same Kubernetes version, and applying it still replaces the VMs. The air-gap bundle uses the product version (`v2.17.1` in the 2.17 bundle this repo describes).

Platform apps move with the product version, as the set in `kommander-applications-vX.Y.Z`. Catalog apps keep their own chart versions. Bundled CNI and CSI move to the build that NKP release ships, via `nkp upgrade addons` or the Nutanix cluster upgrade.

On a pre-provisioned cluster the host OS is the OS vendor's package version, and the kubelet is the Kubernetes version Cluster API wrote. Those two move separately because Cluster API does not replace the disk.

AOS `7.5` / `7.6` and Prism Central `pc.7.5` are infrastructure prerequisites of an NKP release, recorded beside the cluster version. Source: [NKP 2.19](https://next.nutanix.com/product-updates/nutanix-kubernetes-platform-nkp-2-19-45966).

```bash
nkp version
kubectl get kubeadmcontrolplane -A -o custom-columns=NAME:.metadata.name,VERSION:.spec.version
```

## GKE

`1.35.6-gke.1641000`

| Field | Example | What it is |
| --- | --- | --- |
| Kubernetes major.minor.patch | `1.35.6` | Upstream Kubernetes. The patch digit tracks an upstream patch. |
| GKE build | `-gke.1641000` | Google's build of that patch. It can contain a GKE fix, a node image rebuild, or both. |

Google publishes several `-gke.N` builds for one Kubernetes patch. The versioning page's own example is `1.35.6-gke.1638000`, `1.35.6-gke.1641000`, and `1.35.6-gke.1250000`, so the integer is an identifier and the release order is the list `gcloud container get-server-config` returns for the channel.

Aliases, on create and on upgrade:

| You pass | What GKE selects |
| --- | --- |
| `1.35.6-gke.1641000` | That build. |
| `1.35.6` | The highest `-gke.N` GKE currently offers for that patch. |
| `1.35` | The highest patch and `-gke.N` in that minor. |
| `latest` | The highest version in that location. It does not keep the cluster there. |
| `-` | On a node-pool upgrade, the control plane's version. |

The node image type (`COS_CONTAINERD`, `UBUNTU_CONTAINERD`, Windows) is a separate field. Both Linux types use the same `1.Y.Z-gke.N` version. Container-Optimized OS has its own milestone; the milestone is a property of the GKE build you picked. A control plane and a node pool each store this version, and they may differ inside the skew window.

During standard support a `-gke.N` can carry features, bug fixes, and security fixes, and a new one shows up about weekly. After the minor enters extended support, further builds on the `extended` channel are security fixes. The channel is a separate field. Channels decide which of these strings auto-upgrade will aim at. The scheme is [GKE versioning](https://docs.cloud.google.com/kubernetes-engine/versioning).

```bash
gcloud container clusters describe "$CLUSTER" --location "$REGION" \
  --format='value(currentMasterVersion,currentNodeVersion)'
gcloud container node-pools describe "$POOL" --cluster "$CLUSTER" --location "$REGION" \
  --format='value(version,config.imageType)'
```

## EKS

Four numbers. They move on different days.

| Number | Example | Who moves it | What a bump contains |
| --- | --- | --- | --- |
| Cluster minor | `1.36` | You, with `update-cluster-version` | The next Kubernetes minor. One step. |
| Platform version | `eks.13` | AWS, automatically, for the minor you are on | The control-plane Kubernetes patch, plus control-plane fixes that sometimes share a patch. `1.36.4` is `eks.13`. `1.36.2` shipped as `eks.6` through `eks.10`. |
| AMI release | `1.36.5-20261107` | You, with `update-nodegroup-version` or Karpenter drift | Kubelet patch, kernel, containerd. The date is the AMI build. |
| Add-on | `v1.20.3-eksbuild.1` | You, with `update-addon` | That add-on's upstream version, then AWS's rebuild. |

`describe-cluster` shows `version` (`1.36`) and `platformVersion` (`eks.13`). `describe-cluster-versions` also shows `kubernetesPatchVersion`, which is the patch the current platform version is on. A new minor starts at `eks.1`. Later platform versions are `eks.2`, `eks.3`, and up, and AWS skips numbers it discards internally. Platform versions of `1.36` and `1.35` are unrelated counters. AWS rolls existing clusters forward inside the minor. `update-cluster-version` is how you leave the minor. A control-plane CVE on the minor you already run arrives as the next `eks.N`.

The AMI name and the AMI release version put the date in different places:

```text
amazon-eks-node-al2023-x86_64-standard-1.36-v20261107
                                        │    │
                                        │    └─ AMI release date
                                        └─ Kubernetes minor

1.36.5-20261107
│      │
│      └─ same date
└─ kubelet patch inside that AMI
```

A later date at the same `1.36.5` is an OS and userspace rebuild of that kubelet patch. A new kubelet patch is a new `1.36.6-YYYYMMDD` and a new date. Every patch of a minor is compatible with that minor's control plane, so the node AMI can lag or lead the control plane's patch inside the minor. Bottlerocket and Windows AMIs are published per minor with their own version strings; the date rule above is the Amazon Linux AMI. Source: [platform versions](https://docs.aws.amazon.com/eks/latest/userguide/platform-versions.html), [AMI versions](https://docs.aws.amazon.com/eks/latest/userguide/eks-linux-ami-versions.html).

Add-on versions are `v<upstream>-eksbuild.<n>`:

| Piece | Example | What it counts |
| --- | --- | --- |
| Upstream | `v1.20.3` | That project's release. VPC CNI, EBS CSI, EFS CSI, and CoreDNS use their own semver. `kube-proxy` uses the Kubernetes patch (`v1.36.4-eksbuild.N`). |
| `eksbuild.N` | `eksbuild.1` | AWS's rebuild of that upstream. `eksbuild.2` is a new image of `v1.20.3` (a CVE in a dependency, packaging). A new upstream starts again at `eksbuild.1`. |

`describe-addon-versions` is the list of builds for a minor. The cluster version does not select one for you. Extended support keeps publishing platform versions, AMI dates, and add-on builds for that minor for 12 months after the 14 months of standard support. The minor string stays `1.31`. Source: [Kubernetes versions on EKS](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html), [add-on compatibility](https://docs.aws.amazon.com/eks/latest/userguide/addon-compat.html).

```bash
aws eks describe-cluster --name "$CLUSTER" \
  --query 'cluster.{version:version,platform:platformVersion}'
aws eks describe-addon --cluster-name "$CLUSTER" --addon-name vpc-cni \
  --query 'addon.addonVersion'
```

## EKS Anywhere

Two fields on the cluster, plus an image URL.

| Field | Example | Cadence |
| --- | --- | --- |
| `spec.eksaVersion` | `v0.26.4` | Product. Minor about every 4 months (`0.25` on 1 March 2026, `0.26` on 16 July 2026). Patch about every two weeks on the newest minor. The previous minor gets monthly CVE and bug-fix patches. The major is still 0. |
| `spec.kubernetesVersion` | `1.36` | A Kubernetes minor. `0.26` offers 1.30 through 1.36. The Kubernetes patch is the one that `v0.26.x` bundled. |
| OS image | template name, or `osImageURL` | Built per `kubernetesVersion`. A new URL at the same minor is the OS rebuild. |

A CVE in Kubernetes or in the EKS Anywhere components is a new `v0.26.x`. You install that CLI, set `eksaVersion`, and upgrade. The machines roll. Cilium's version is the one in that EKS Anywhere release. Curated packages stay on `spec.packageVersion` until you change that field. A workload cluster's `eksaVersion` may trail the management cluster by at most one product minor.

Extended support for a Kubernetes minor is 12 months after the 14 months of standard support, for clusters with a license token. The patch still arrives as an EKS Anywhere patch release. Source: [EKS Anywhere version lifecycle](https://anywhere.eks.amazonaws.com/docs/concepts/support-versions/).

## AKS

Three fields. A cluster upgrade can move the first one, the second one, or the third one.

| Field | Example | Where it lives |
| --- | --- | --- |
| Control plane | `1.32.3` | Managed cluster `kubernetesVersion`. Full patch. |
| Kubelet | `1.32.3` | Agent pool `orchestratorVersion`. It may lag the control plane. |
| Node image | `AKSUbuntu-2404gen2containerd-202604.07.0` | Agent pool `nodeImageVersion`. The Kubernetes version for that pool is `orchestratorVersion`. |

The node image name:

```text
AKSUbuntu-2404gen2containerd-202604.07.0
│         │    │   │         │
│         │    │   │         └─ VHD release: YYYYMM.DD.revision
│         │    │   └─ container runtime
│         │    └─ VM generation
│         └─ Ubuntu 24.04
└─ distro (AKSUbuntu, AKSAzureLinux, …)
```

`202604.07.0` is the weekly VHD: year and month, day, revision. It carries OS fixes and bug fixes at the current orchestrator version. Windows images use the Windows Server build and a date, for example `20348.2113.260115`.

The `SecurityPatch` channel adds a second date when it rolls a security-only VHD:

```text
AKSUbuntu-2204gen2containerd-202410.27.0-2024.12.01
                                      │            │
                                      │            └─ security-patch date
                                      └─ base VHD
```

That suffix is CVE fixes from the image maintainer, without the weekly VHD's bug fixes. A security fix that AKS can apply live does not change the VHD and does not change this label. The label moves when a new VHD is what got applied. On the `Unmanaged` channel the label stays on the VHD AKS allocated, while the guest applies its own packages; the label is then the image AKS handed you, and the package set is whatever the guest has installed since.

The cluster auto-upgrade channel and the node OS channel are different knobs on these fields. `patch` moves `1.32.0` to `1.32.3` and leaves the minor. `stable` and `rapid` take a new minor on their schedules. `NodeImage` moves the VHD date. LTS adds a year after community support, about 24 months from GA, on the same `1.Y.Z`. During that year the two newest patches are the supported ones. Source: [node OS channels](https://learn.microsoft.com/en-us/azure/aks/auto-upgrade-node-os-image), [node images](https://learn.microsoft.com/en-us/azure/aks/node-images), [LTS](https://learn.microsoft.com/en-us/azure/aks/long-term-support).

```bash
az aks show -g "$RG" -n "$CLUSTER" --query kubernetesVersion -o tsv
az aks nodepool list -g "$RG" --cluster-name "$CLUSTER" \
  --query '[].{name:name,k8s:orchestratorVersion,image:nodeImageVersion}' -o table
```

## RKE2 and Rancher

RKE2 is the Kubernetes version plus a rebuild counter. Rancher is a different product version that certifies a range of those.

`v1.32.3+rke2r2`

| Field | Example | What a bump is |
| --- | --- | --- |
| Kubernetes | `v1.32.3` | Upstream major, minor, and patch. RKE2 aims to ship a patch within about a week of upstream and a minor within about 30 days. |
| Rebuild | `+rke2r2` | Semver build metadata. `r1` is the first RKE2 release of that Kubernetes patch. `r2` is another release of the same patch: a bundled chart, containerd, a hardened image, or a CVE that is not a Kubernetes patch. |
| Pre-release | `v1.32.3-rc1+rke2r1` | A candidate. The `testing` channel. |

Channels (`stable`, `latest`, `testing`, `v1.32`) pick a moving target. `INSTALL_RKE2_VERSION` records the concrete string, `v1.Y.Z+rke2rN`. Pinning the minor channel `v1.32` follows later patches of that minor. The RPM tags `v1.32.3+rke2r1.stable.0` are packaging revisions; the version the node runs is the part before `.stable`.

Bundled components print their own versions in the release notes, with a SUSE rebuild suffix: `etcd v3.5.19-k3s1`, `containerd v2.0.4-k3s2`, `ingress-nginx v1.12.1-hardened1`. The `-k3sN` or `-hardenedN` suffix is a rebuild of that upstream, the same idea as `eksbuild`. Those versions move when `+rke2rN` moves. The host kernel and the OS packages do not. `rpm -q kernel` is the OS version. `rke2 --version` will not show it.

Rancher is `v2.15.3`.

| Field | Example | What it governs |
| --- | --- | --- |
| Major | `2` | The Rancher 2 line. |
| Minor | `15` | The support matrix. Rancher 2.15.3 certifies downstream Kubernetes 1.34 through 1.36 for RKE2, k3s, AKS, EKS, and GKE. |
| Patch | `3` | The manager. Downstream RKE2 versions stay where you set them. `rancher-monitoring` and `rancher-logging` stay until you upgrade those charts. |

Fleet's agent chart has its own version and its own Kubernetes bounds. A Rancher patch leaves that chart where it is. Source: the [RKE2 release cadence](https://github.com/rancher/rke2#release-cadence), and the Rancher matrix cited in [Compatibility](compatibility.md).

```bash
rke2 --version
kubectl get nodes -o custom-columns=NAME:.metadata.name,KUBELET:.status.nodeInfo.kubeletVersion
```

## What to write down

Record every number that can move on its own. A single "the cluster is 1.36" hides the one that is stale.

| Platform | The set |
| --- | --- |
| OpenShift | `4.y.z`, and `machine-os` from `oc adm release info`. CSV versions for operators you subscribed to. |
| NKP | Product `2.y.z`, the image name, the Kubernetes version on `KubeadmControlPlane`. |
| GKE | Control plane `1.Y.Z-gke.N`, each pool's `1.Y.Z-gke.N`, image type. |
| EKS | Minor, `eks.N`, AMI release `1.Y.Z-YYYYMMDD`, each add-on's `vX.Y.Z-eksbuild.N`. |
| EKS Anywhere | `eksaVersion`, `kubernetesVersion`, the OS template or `osImageURL`, each `packageVersion`. |
| AKS | `kubernetesVersion`, each pool's `orchestratorVersion`, each pool's `nodeImageVersion` including a security-patch suffix. |
| RKE2 | `v1.Y.Z+rke2rN` on servers and on agents, the OS package version, your chart versions. Rancher `v2.y.z` on the manager. |
