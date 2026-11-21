# Compatibility

How far behind a component may sit, and how far ahead the management plane may move before the cluster catches up. Both are support windows. A window is not a CVE backport.

Documented November 2026. The linked page is the proof. Patch releases extend a row, and a vendor page can be stricter than the upstream page it cites. Read the page for the minor you are about to install, then dry-run.

Two directions:

| Direction | What it allows | What people mix it up with |
| --- | --- | --- |
| Backward | The older binary is still inside the range the newer one publishes. A kubelet three minors behind a 1.28+ apiserver. A workload cluster on a Kubernetes minor that this Cluster API release still lists. An add-on version whose compatibility list still contains this cluster minor. | A reason to stay there. The old binary keeps the old CVEs. |
| Future | The newer component is allowed to move first, and its release already lists the version you are about to install. Cluster API on the management cluster, upgraded before the workload Kubernetes version. Platform apps, upgraded before the cluster minor they must understand. An EKS add-on version that already lists the target minor. Flux on the management cluster, newer than the Flux still running on the workload cluster. | A belief that the newer copy is the one serving the cluster. It is not. The cluster copy pulls, issues tokens, and answers queries. See [two copies](#two-copies-management-is-ahead). |

Upstream is the widest window. Every product on this site narrows it. ClusterClass clusters narrow it to one Kubernetes version for every machine.

## Upstream Kubernetes

Source: [Kubernetes version skew policy](https://kubernetes.io/releases/version-skew-policy/). The project maintains the latest three minors. In November 2026 those are 1.37, 1.36, and 1.35, which is also what [endoflife.date/kubernetes](https://endoflife.date/kubernetes) shows as still maintained. Examples below use an apiserver at 1.37, which is the example on the skew page. The dated calendar is the next section.

| Component | May be older by | May be newer by | At apiserver 1.37 |
| --- | --- | --- | --- |
| Other `kube-apiserver` instances in an HA cluster | 1 minor | 1 minor, and the newest and oldest stay within one minor of each other | 1.37 and 1.36 |
| `kubelet` | 3 minors. A kubelet older than 1.25 is capped at 2. | None. The kubelet is at or below the apiserver. | 1.37, 1.36, 1.35, 1.34 |
| `kube-proxy` vs apiserver | 3 minors (2 if kube-proxy is older than 1.25) | None | 1.37, 1.36, 1.35, 1.34 |
| `kube-proxy` vs the kubelet beside it | 3 minors | 3 minors | — |
| `kube-controller-manager`, `kube-scheduler`, `cloud-controller-manager` | 1 minor, for the duration of the live upgrade | None | 1.37 and 1.36 |
| `kubectl` | 1 minor | 1 minor | 1.38, 1.37, 1.36 |

HA narrows the rest. If one apiserver is still 1.36 while another is 1.37, a 1.37 kubelet is already too new for the 1.36 apiserver, and kubectl is supported only at 1.37 and 1.36.

The supported component order on that same page is: newest patch of the current minor, then `kube-apiserver`, then controller-manager / scheduler / cloud-controller-manager in any order, then kubelet and kube-proxy. One minor at a time. The n-3 kubelet window is how long workers may lag during that walk. It is not a license to move the apiserver three minors in one change.

kubeadm enforces this. Cluster API's kubeadm bootstrap provider follows the kubeadm skew policy, and an image built by image-builder puts one Kubernetes version on the machine. Both are called out on the [Cluster API version page](https://cluster-api.sigs.k8s.io/reference/versions.html).

## Upstream patch calendar

Skew says a 1.34 kubelet may still talk to a 1.37 apiserver. The patch calendar says 1.34 stopped receiving fixes on 27 Oct 2026. Both are true. A kubelet inside the skew window and past maintenance keeps every CVE that landed only on the three maintained branches.

Kubernetes patches the latest three minors (N-2) on a cycle of about 15 weeks. [endoflife.date/kubernetes](https://endoflife.date/kubernetes) records that as about 14 months on a minor: 12 months of active support, then about 2 months of maintenance as the upgrade period. Active support is the date they label Active Support. Maintenance support is the date they label end of life. The policy those dates restate is the [Kubernetes patch releases](https://kubernetes.io/releases/patch-releases/) page. The table below is the endoflife.date API as generated 21 Nov 2026 (page last modified 29 Oct 2026). Run `kubectl version` and match the minor to a row that is still maintained, on the latest patch in that row.

| Minor | Released | Active support ends | Maintenance ends | Latest patch | Patches in Nov 2026 |
| --- | --- | --- | --- | --- | --- |
| 1.37 | 26 Aug 2026 | 28 Aug 2027 | 28 Oct 2027 | 1.37.2 (27 Oct 2026) | Yes |
| 1.36 | 22 Apr 2026 | 28 Apr 2027 | 28 Jun 2027 | 1.36.6 (27 Oct 2026) | Yes |
| 1.35 | 17 Dec 2025 | 28 Dec 2026 | 28 Feb 2027 | 1.35.10 (27 Oct 2026) | Yes. Active support ends 28 Dec 2026. |
| 1.34 | 27 Aug 2025 | 27 Aug 2026 | 27 Oct 2026 | 1.34.13 (27 Oct 2026) | No. 1.34.13 is the final patch. |
| 1.33 | 23 Apr 2025 | 28 Apr 2026 | 28 Jun 2026 | 1.33.13 (11 Jun 2026) | No |
| 1.32 | 11 Dec 2024 | 28 Dec 2025 | 28 Feb 2026 | 1.32.13 (26 Feb 2026) | No |
| 1.31 | 13 Aug 2024 | 28 Aug 2025 | 11 Nov 2025 | 1.31.14 (11 Nov 2025) | No |
| 1.30 | 17 Apr 2024 | 28 Apr 2025 | 15 Jul 2025 | 1.30.14 (17 Jun 2025) | No |
| 1.29 | 13 Dec 2023 | 28 Dec 2024 | 28 Feb 2025 | 1.29.15 (11 Mar 2025) | No |

Minors older than 1.29 are on the same page. They have been out of maintenance for more than a year.

OpenShift EUS, EKS extended support, AKS LTS, and a Rancher support matrix can keep a minor after the maintenance date above. Rancher 2.15.3 still certifies downstream 1.34. The vendor date is the one for that product. This table is upstream Kubernetes.

## Control plane and nodes, per platform

| Platform | Nodes may lag the control plane | Nodes may lead | One change may skip a minor | Proof |
| --- | --- | --- | --- | --- |
| Upstream | 3 minors (2 before kubelet 1.25) | No | Apiserver moves one minor | [version skew policy](https://kubernetes.io/releases/version-skew-policy/), [patch calendar](https://endoflife.date/kubernetes) |
| OpenShift | Steady state is equal versions on control-plane hosts and node hosts. During an upgrade, kubelet may be 1 minor older on an odd OpenShift minor, and 2 minors older on an even minor. Kubelet is never newer than the apiserver. | No | One minor, along the Cincinnati graph. An EUS-to-EUS hop uses the even-minor window of 2 so workers can stay on the previous EUS while the control plane walks the odd minor in between. Pause `mcp/worker` only inside that window. | [OpenShift 4.22 control plane architecture](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/architecture/control-plane) |
| NKP, ClusterClass (2.19 topology) | No lasting lag. `Cluster.spec.topology` enforces one Kubernetes version on every machine. | No | One NKP minor, and the Kubernetes minor that release ships. NKP 2.19 installs 1.36.2 by default and still supports the 1.35.x family. | [Cluster API topology rule](https://cluster-api.sigs.k8s.io/reference/versions.html), [NKP 2.19](https://next.nutanix.com/product-updates/nutanix-kubernetes-platform-nkp-2-19-45966) |
| NKP pre-provisioned | You choose. The ceiling is the upstream kubelet window. | No | One minor. The host has to already meet the new kubelet. | Upstream policy, plus the pre-provisioned row in [Cluster API](capi.md) |
| GKE | 2 minors. A 1.31 control plane supports nodes at 1.31, 1.30, or 1.29. Autopilot nodes track the control plane. | No | Control plane, no. A node pool may skip a minor when the result is still inside the two-minor window. Google's own guidance is to bring nodes up to the control plane on each minor instead. | [GKE versioning and skew](https://docs.cloud.google.com/kubernetes-engine/versioning) |
| EKS | 3 minors for managed node groups, Fargate, and self-managed nodes, from Kubernetes 1.28. Before 1.28 the window is 2. | No | One minor | [EKS cluster upgrade practices](https://docs.aws.amazon.com/eks/latest/best-practices/cluster-upgrades.html) |
| EKS Anywhere | 3 minors from Kubernetes 1.28, 2 before that. Separately, the management cluster's EKS Anywhere version may be at most one minor ahead of a workload cluster, and a workload cluster may not be ahead of the management cluster. Management components and cluster components have the same one-minor cap. | Workload EKS Anywhere version, no. Workload Kubernetes version, no, relative to its own control plane. | EKS Anywhere minors, no (`0.23` to `0.25` is unsupported). Kubernetes minors, no. | [Upgrade overview](https://anywhere.eks.amazonaws.com/docs/clustermgmt/cluster-upgrades/upgrade-overview/), [version lifecycle](https://anywhere.eks.amazonaws.com/docs/concepts/support-versions/) |
| AKS | 3 minors from Kubernetes 1.28. A 1.35 control plane supports pools at 1.35, 1.34, 1.33, and 1.32. The control plane is equal to or newer than every pool. | No | Non-LTS clusters, no. An LTS cluster may skip minors when the landing versions still satisfy the skew check. | [Control plane upgrade rules](https://learn.microsoft.com/en-us/azure/aks/upgrade-aks-control-plane), [supported versions](https://learn.microsoft.com/en-us/azure/aks/supported-kubernetes-versions) |
| RKE2 | The upstream window. Servers land on the new version before agents. | No | No. The installer will not stop you. The skew policy is the limit. | [RKE2 manual upgrades](https://docs.rke2.io/upgrades/manual) |
| Rancher 2.15.3 | The manager is certified against downstream Kubernetes 1.34 through 1.36 for RKE2, k3s, AKS, EKS, and GKE. 2.15 dropped 1.33. The oldest certified cluster is two minors behind the newest the manager knows. | A downstream minor newer than 1.36 is outside this matrix. | Upgrade Rancher onto the minor that lists the Kubernetes version you want, then upgrade the cluster one minor at a time. | [Rancher 2.15.3 support matrix](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-15-3/) |

EKS Anywhere publishes the Kubernetes minors inside each product version. In November 2026 both of these were still receiving patches:

| EKS Anywhere | Released | Kubernetes minors in that build |
| --- | --- | --- |
| 0.26 | 16 July 2026 | 1.36, 1.35, 1.34, 1.33, 1.32, 1.31, 1.30 |
| 0.25 | 1 March 2026 | 1.35, 1.34, 1.33, 1.32, 1.31, 1.30, 1.29 |

A Kubernetes version stays under standard support for 14 months after it appears in EKS Anywhere, then 12 months of extended support with a license token, from v0.22. Creating a cluster on a Kubernetes minor the EKS Anywhere version does not list is outside this table even when that minor still exists upstream. Source: [EKS Anywhere version lifecycle](https://anywhere.eks.amazonaws.com/docs/concepts/support-versions/).

Rancher 2.14.7, the previous minor, is certified for downstream Kubernetes 1.33–1.35. Apply the row for the manager you are actually running. Source: [Rancher 2.14.7 support matrix](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-14-7/).

AKS has a stale sentence. [Upgrade node pools](https://learn.microsoft.com/en-us/azure/aks/upgrade-node-pools) still says the pool must be within two minors. The control-plane page, the supported-versions FAQ, and the API error `NodePoolMcVersionIncompatible` ("cannot be more than 3 versions less") are the current rule. When those disagree, believe the error the API returns. See [AKS upgrade blocked](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/error-codes/aks-upgrade-blocked).

GKE's two-minor window is the product rule. Upstream's three-minor window does not raise it. A pool that falls out is reported as `CLUSTER_VERSION_SKEW_UNSUPPORTED`.

## Cluster API

Source: [Version support](https://cluster-api.sigs.k8s.io/reference/versions.html). Checked against the [v1.12.0](https://github.com/kubernetes-sigs/cluster-api/releases/tag/v1.12.0), [v1.13.0](https://github.com/kubernetes-sigs/cluster-api/releases/tag/v1.13.0), [v1.14.0](https://github.com/kubernetes-sigs/cluster-api/releases/tag/v1.14.0), and [v1.14.3](https://github.com/kubernetes-sigs/cluster-api/releases/tag/v1.14.3) release notes.

A minor, on the day it is cut, supports four Kubernetes minors on the management cluster (N through N-3) and six on workload clusters (N through N-5). Later patch releases try to add the next Kubernetes minor. The project says the backport is an attempt, not a guarantee. From Kubernetes 1.36 onward they also try to extend the previous CAPI minor.

Workload clusters may be older than the management cluster. That is the backward direction, and it is why the workload column is wider. A workload cluster may not run a Kubernetes minor the release does not list. That is the future limit: upgrade the management controllers onto a CAPI release that lists the target, and only then roll the workload.

A self-hosted management cluster (NKP `--self-managed`, EKS Anywhere management cluster) is held to the management column, which drops the oldest workload-only rows.

| CAPI | Where it is in November 2026 | Management cluster | Workload cluster |
| --- | --- | --- | --- |
| v1.14 | Standard support. Maintenance starts when v1.16.0 ships. EOL when v1.17.0 ships. | 1.33–1.36 on v1.14.0. 1.37 from v1.14.1. | 1.31–1.36 on v1.14.0. 1.37 from v1.14.1. |
| v1.13 | Standard support. Maintenance starts when v1.15.0 ships. EOL when v1.16.0 ships. | 1.32–1.35 on v1.13.0. 1.36 from v1.13.1. 1.37 from v1.13.6. | 1.30–1.35 on v1.13.0, then the same patch floors through 1.37. |
| v1.12 | Maintenance since v1.14.0 on 2026-08-18. EOL when v1.15.0 ships. | 1.31–1.34 on v1.12.0. 1.35 from v1.12.1. 1.36 from v1.12.8. 1.37 is absent. | 1.29–1.34 on v1.12.0, then the same patch floors through 1.36. |
| v1.11 and older | EOL. v1.11 ended 2026-08-18. | — | — |

A checkmark on the upstream table means both planes, unless the cell says "only workload". v1.12 can still create a 1.29 workload cluster. It cannot run its own controllers on a 1.29 management cluster. "Can create" is the whole of that promise for a Kubernetes minor past the [patch calendar](#upstream-patch-calendar). 1.29 maintenance ended 28 Feb 2025. Cluster API can start that cluster and can upgrade it to the next minor. Kubernetes CVEs on 1.29 are not part of the promise.

Other rules from the same page:

- Providers installed on one management cluster implement that release's contract. CAPX, CAPA, CAPV, and the rest do not have to wear the same version number as core CAPI. `clusterctl` is what lines the contracts up. Upgrade `clusterctl` first. NKP wraps that step in `nkp upgrade capi-components` (included in the Nutanix management-cluster upgrade).
- CAPI itself upgrades at most n-3 minors in one hop (v1.11 to v1.14 is the far edge). Downgrades are unsupported.
- v1beta1 is deprecated since v1.11 and is planned to stop being served in v1.16, April 2027. v1beta2 is the contract current releases speak. A core provider on v1beta2 can temporarily sit next to an infrastructure provider still on v1beta1.
- Kubeadm config API is v1beta4 from Kubernetes 1.31, and v1beta3 from 1.22 through 1.30.
- On a ClusterClass cluster, one Kubernetes version applies to every machine. Control-plane-ahead-of-workers, which GKE, EKS, and AKS allow inside their windows, is not a ClusterClass state.

NKP and EKS Anywhere ship a CAPI build. Read the version off the management cluster. Do not install a newer upstream CAPI beside the product's controllers.

```bash
kubectl get deploy -A | grep -E 'capi-controller-manager|capx-controller-manager|capa-controller-manager|capv-controller-manager|capz-controller-manager'
kubectl get crd clusters.cluster.x-k8s.io -o jsonpath='{.spec.versions[*].name}{"\n"}'
```

The image tag is the CAPI minor. The CRD `served` versions tell you whether v1beta1 is still accepted. Match the minor to the table above, including the patch floor, before you declare a workload Kubernetes version.

## Add-ons

An add-on release publishes a set of Kubernetes minors. Backward means this cluster's minor is in the set. Future means the target minor is already in the set, so you can move the add-on before the cluster, or in the same change window, and the new cluster will still accept it.

| Add-on | Backward | Future | Proof |
| --- | --- | --- | --- |
| EKS managed add-on (`vpc-cni`, `kube-proxy`, `coredns`, `aws-ebs-csi-driver`, CloudWatch, ADOT) | `describe-addon-versions` returns that `addonVersion` with this minor in `compatibilities[].clusterVersion`. The control-plane upgrade leaves the add-on where it is. The old build keeps working until its row disappears for the new minor. | Query the target minor, not the current one. A version is future-compatible only when the target minor is in that list. `defaultVersion: true` is the one AWS would install today. | [Add-on compatibility](https://docs.aws.amazon.com/eks/latest/userguide/addon-compat.html) |
| GKE Dataplane V2, GCE PD CSI, managed Prometheus, logging collectors | They track the node image or the control plane. There is no older pin that stays supported across a minor. | They arrive with the GKE upgrade. A chart you installed stays on the chart version you installed. | [GKE](platforms/gke.md) |
| OpenShift payload operators (network, monitoring, ingress, authentication, image registry) | They move with the payload. Holding one of them on the previous minor is not a supported state. | The next catalog index (`v4.18` to `v4.19`, and the same for later minors) is the future. An optional operator needs a CSV in that index. | [OpenShift](platforms/openshift.md) |
| OpenShift optional operators (logging, GitOps, Service Mesh, ODF, a cloud CSI subscription) | The installed CSV has to remain valid for the minor you are on. | Approve an InstallPlan whose CSV exists for the target minor before `oc adm upgrade`, or the subscription sits `UpgradePending` when the index flips. | [OpenShift](platforms/openshift.md) |
| NKP core add-ons (CNI, bundled CSI) | The version in this NKP release. A newer upstream Cilium or CSI than the release is outside support. | `nkp upgrade addons`, or the Nutanix cluster upgrade which includes them, in the documented order. | [NKP](platforms/nkp.md) |
| NKP platform apps | The previous release's apps are not compatible with the Kubernetes version in the next NKP release. They move first for that reason. | `nkp upgrade kommander` moves the management copies. `nkp upgrade workspace` (through 2.18) or `nkp upgrade platform-apps` (2.19, non-ClusterClass) moves each cluster's copies up to the management version. Until that second step, the cluster copy is the old chart on purpose. | The upgrade guide states both: platform apps from the earlier NKP version are not compatible with the Kubernetes in the current version, and the workspace upgrade brings cluster apps to the same version as the management cluster. [NKP 2.19](https://next.nutanix.com/product-updates/nutanix-kubernetes-platform-nkp-2-19-45966) |
| cert-manager | 1.20 (10 Mar 2026): Kubernetes 1.32–1.35, OpenShift 4.19–4.21. 1.21 (8 Jul 2026): Kubernetes 1.33–1.36, OpenShift 4.20–4.22. The project keeps at least two minors. 1.20 ends when 1.22 ships. | Move onto a release whose range contains the target Kubernetes minor before you leave the old range. | [cert-manager supported releases](https://cert-manager.io/docs/releases/) |
| Flux | Each Flux minor supports the latest three Kubernetes minors. Flux 2.10 (26 Oct 2026): 1.35.2+, 1.36.1+, 1.37.0+. Flux 2.9 (30 Jun 2026): 1.34.1+, 1.35.0+, 1.36.0+. The project supports the latest three Flux minors (2.10, 2.9, 2.8 in this window; 2.7 is end of life). | Flux 2.10 is how you land on 1.37. It does not extend backward to 1.34. Controllers inside one cluster stay within one Flux minor of each other. The `flux` CLI stays within one Flux minor of the controllers. Any v2 patch upgrades to any later v2 patch. | [Flux releases](https://fluxcd.io/flux/releases/), [v2.10.0](https://github.com/fluxcd/flux2/releases/tag/v2.10.0), [v2.9.0](https://github.com/fluxcd/flux2/releases/tag/v2.9.0) |
| A chart you installed | `kubeVersion` on the chart, and the APIs it calls on this minor. | Bump it onto a release that lists the target minor before the cluster upgrade when it calls an API the next minor removes. The platform upgrade will not do this. | The chart's own artifact hub or release notes |

```bash
aws eks describe-addon-versions \
  --addon-name vpc-cni \
  --kubernetes-version 1.36 \
  --query 'addons[].addonVersions[].{version:addonVersion,clusters:compatibilities[].clusterVersion,default:compatibilities[].defaultVersion}'
```

Run that for the minor you are on and for the minor you want. The second query is the future check. An empty list means that add-on build is not future-compatible with the target, however long it has been healthy on the current minor.

## Two copies: management is ahead

After the management plane moves, and before that cluster's platform apps move, the same app is a higher version on the management cluster than on the workload cluster. That gap is the upgrade sequence. `nkp upgrade kommander` finishes before `nkp upgrade workspace` or `nkp upgrade platform-apps`. Rancher's manager minor finishes before each downstream feature chart. An OpenShift hub's payload finishes before a managed cluster's.

The copy on the workload cluster is the one that pulls, serves, and answers. The management copy being newer does not patch the cluster copy, and a CVE fixed only in the management image is still present on the cluster.

### Flux

Two installs.

The management install (`kommander-flux` on NKP, `flux-system` on a fleet repo) reconciles sources on the management cluster. It also applies the manifests that install or upgrade Flux on each workload cluster: a Kommander platform-app rollout, or a Flux `Kustomization` with `spec.kubeConfig` aimed at that cluster's kubeconfig.

The workload install reconciles `GitRepository`, `OCIRepository`, `HelmRelease`, and `Kustomization` objects that live on that cluster. Those objects are pulled by the workload controllers. A `HelmRelease` applied to a cluster that has no Flux fails with `no matches for kind`, because the CRD and the controller are both absent. That is [fluxcd/flux2 discussion 3280](https://github.com/fluxcd/flux2/discussions/3280). Hosting the `HelmRelease` on the management cluster and setting `kubeConfig` is a different layout: the management controller then applies the rendered chart to the remote API. NKP's platform apps are the first layout. The cluster's own Flux is what you see reconciling `AppDeployment`s there.

So a higher `kommander-flux` image on the management cluster, beside an older Flux image on the workload cluster, is the normal state between the two upgrade steps. The workspace upgrade exists to bring the cluster apps up to the management version. Until it runs, they differ.

Flux's one-minor controller rule applies inside one cluster. Management at 2.10 and a workload at 2.8 do not share an API server, so that pair is outside the rule. Installing 2.10 controllers next to 2.8 controllers in the same cluster is inside the rule and outside the guarantee. Installing 2.10 onto a cluster still on Kubernetes 1.34 is outside the 2.10 support table even when the management cluster is already on 1.36.

A hand edit of the workload `HelmRelease` image is put back by the next platform upgrade. On a GitOps fleet it is put back by whatever path installs Flux on that cluster.

### Dex

Dex on the management cluster is the platform login broker. It moves with Kommander, or with whichever chart installs the management IdP.

Where the platform set also installs Dex on the workload cluster, that Deployment stays on the last platform-app version until the workspace or platform-app upgrade. The workload kube-apiserver trusts the issuer it was configured with: `--oidc-issuer-url` and the related flags, or an `AuthenticationConfiguration` from Kubernetes 1.30. It verifies ID tokens with the keys published by that issuer. [Dex's Kubernetes guide](https://dexidp.io/docs/guides/kubernetes/) is the flow. The apiserver does not follow the management cluster's Dex version.

A newer Dex on the management cluster changes login for the management UI. It leaves the workload cluster's Dex Deployment, and it leaves the issuer URL that workload apiserver already has. Rotating the management signing key or issuer URL logs management users out. It does not roll the cluster-local Dex.

Dex does not publish a Kubernetes minor matrix the way Flux does. The supported pair is the issuer the apiserver is configured with and the Dex that serves that issuer.

### Thanos

`kube-prometheus-stack` on the management cluster carries the Querier, Query Frontend, Store Gateway, and Compactor. That chart moves with the management platform upgrade. A 2.18 workspace upgrade prints the pair (`kube-prometheus-stack-78.4.0` to `kube-prometheus-stack-82.13.6` is the shape the command shows).

Each workload cluster keeps the Prometheus and the Thanos Sidecar from its own copy of that chart until its platform apps move. The sidecar uploads blocks and serves StoreAPI for data still in the local Prometheus. The central Querier is what Grafana on the management cluster talks to. Those are different versions on purpose during the gap.

Thanos treats the StoreAPI as backward compatible when it can, and 0.y releases are allowed to break it. The changelog marks those lines. The project discussion is [thanos#4068](https://github.com/thanos-io/thanos/discussions/4068): components can move in any order while the gRPC API holds, and a break is written down because 0.y has no compatibility promise. One recorded break is [thanos#8729](https://github.com/thanos-io/thanos/issues/8729): Query v0.41.0 panics against a sidecar at v0.34.1, and the sidecar has to be at least v0.35.1. The management Querier can be newer. It cannot be arbitrarily newer. Read the changelog between the two image tags before you take the central chart across a line marked breaking, and move the cluster sidecars first when the notes say the old sidecar cannot answer.

The management Thanos version is not the version that scraped the workload cluster.

### The same gap on the other apps

| App | Higher copy, and when it moves | Copy that serves the cluster | Proof to compare |
| --- | --- | --- | --- |
| Flux | Management `kommander-flux` / `flux-system`, with `nkp upgrade kommander` or the fleet repo | Workload Flux controllers, pulling that cluster's sources | Controller image on each context |
| Dex | Management IdP, with the management upgrade | The issuer that cluster's apiserver is configured to trust. Often a Dex Deployment in the cluster's platform set. | Deployment image, plus the apiserver issuer URL |
| Thanos | Querier, Store, Compactor inside the management `kube-prometheus-stack` | Sidecar and Prometheus in the workload chart | Image tag on the querier Deployment and on the workload sidecar |
| cert-manager, Traefik, gatekeeper | Management platform set | The `AppDeployment` / `HelmRelease` on that cluster | `kubectl get helmreleases -A` on each context |
| Rancher monitoring and logging | The Rancher manager minor. The feature charts do not move with it. | `rancher-monitoring` and `rancher-logging` on the downstream cluster, after you upgrade those charts | Installed App version vs the Rancher version |
| Fleet | Fleet controller, with the Rancher minor. The 2.15 agent chart bounds Kubernetes to `>=1.34 <1.37` and Rancher to `>=2.15 <2.16`. | `fleet-agent` on the downstream cluster, which applies the bundles | Agent chart version vs the manager |
| OpenShift GitOps, and any OperatorHub install | The hub subscription, when you approve that InstallPlan | The spoke subscription. The hub payload does not walk spoke CSVs. | `oc get csv` on each cluster |
| EKS Anywhere packages | The management controllers, at most one EKS Anywhere minor ahead, roll machines | The `Package` object on the workload cluster. `upgrade cluster` does not change `spec.packageVersion`. | `kubectl get packages -A` |

## Prove the versions you are actually running

```bash
# this cluster: apiserver vs kubelet
kubectl version
kubectl get nodes -o custom-columns=NAME:.metadata.name,KUBELET:.status.nodeInfo.kubeletVersion,PROXY:.status.nodeInfo.kubeProxyVersion

# Flux, Dex, Thanos image tags. Run once per context.
kubectl get deploy,ds,sts -A \
  -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.kind}{"/"}{.metadata.name}{"\t"}{range .spec.template.spec.containers[*]}{.image}{" "}{end}{"\n"}{end}' \
  | grep -Ei 'flux|dex|thanos|prometheus'

# which Flux is pulling on the workload cluster
kubectl --context workload get helmreleases,kustomizations,gitrepositories,ocirepositories -A
kubectl --context workload get deploy -A -l app.kubernetes.io/part-of=flux

# NKP: management apps vs the workspace's apps
kubectl --context mgmt get helmreleases -n kommander
kubectl --context workload get appdeployments -A
kubectl --context workload get helmreleases -A
```

The management image tag and the workload image tag are supposed to differ until the platform-app step for that cluster has completed. After it has completed, NKP's upgrade brings them to the same platform-app version. A gap that survives `nkp upgrade workspace` or `nkp upgrade platform-apps` is an unfinished `HelmRelease`, not the expected window. Read the conditions before you run the command again.

```bash
kubectl get helmreleases -A
kubectl get appdeployments -A
```

EKS add-on proof is the API output from the previous section, saved next to the change, for the target minor. CAPI proof is the controller image tag matched to the table on the Cluster API version page, including the patch floor. NKP Kubernetes proof is the release's supported-version list (2.19: 1.36.2 by default, and 1.35.x), not a blog's recollection of it.
