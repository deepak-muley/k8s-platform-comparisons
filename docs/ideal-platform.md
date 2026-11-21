# An ideal Kubernetes platform

One page: [Ideal platform, summary](ideal-platform-summary.md).

This is a design, not a product in this repo. It is the shape the other pages are measuring.

A platform engineer can patch the OS, the control plane, the kubelet, CSI, CNI, the cloud controller manager, and apps such as Prometheus or Flux on separate clocks. Apps are declared once on a hub and land on every spoke you select. At a thousand clusters the home screen is still one grid: rings by components, each cell a version.

Speeds are the ones in [Concepts](concepts.md). Skew windows and the upstream patch calendar are the ones in [Compatibility](compatibility.md). A node roll is still a node roll. The design keeps a DaemonSet fix from becoming one, and it refuses to put both on the same button. Out of band does not widen a window: a kubelet may lag the apiserver by at most three minors and may not lead it, the cloud controller manager may lag by one minor during an upgrade and may not lead, and kube-proxy follows the kubelet's window against the apiserver.

## Out of band

A component is out of band when all three hold:

1. It has its own version and its own artifact.
2. A wave can move that version without writing a new version for any other component.
3. The rollout is the smallest unit that component needs. A chart rolls pods. A node image rolls nodes. The control plane rolls the apiserver.

Compatibility is a matrix on the package. The hub evaluates it before a wave starts. A coupling nobody declared does not show up in the middle of a ring.

## Components

| Component | How it rolls | What moves with it |
| --- | --- | --- |
| Node OS (kernel, OS packages) | Speed 3. New image, or a package update on the host, then a drain. | That node. Kubernetes, CNI, CSI, CCM, and apps keep their pins. |
| Container runtime | Speed 3 when containers must restart. Own artifact, so a runtime CVE ships without a kernel bump. | Pods on the node, via drain. Kernel, kubelet, and charts stay. |
| Kubernetes control plane | API update. Workers stay. | apiserver, etcd, scheduler, controller-manager. |
| Kubelet | Speed 3. Its own field on the pool. May lag the apiserver by three minors. May not be newer. | The node. The OS image stays if you did not rebuild it. |
| kube-proxy | Speed 2. Package. May lag the apiserver by three minors. May not be newer. | Its DaemonSet. |
| CoreDNS | Speed 2. Package. Its own chart range, not the Kubernetes version. | Its Deployment. |
| CNI | Speed 2. Package. | Its pods. Existing flows survive a short restart. New pods wait for the DaemonSet. |
| CSI | Speed 2. Package. Controller Deployment and node DaemonSet. | Its pods. The health check is a volume create plus an existing mount. |
| CCM | Speed 2. Package. Stays within one minor behind the apiserver, and never ahead. That window is why it has its own wave: so a control-plane bump can pull it forward without a node roll. | The controller. LoadBalancer and route behavior. |
| Platform apps (Flux, Prometheus, cert-manager, ingress, policy) | Speed 2. One chart each. | That app. The other apps stay. |
| Tenant apps | Speed 2. Same pipe, separate RBAC. | That app on the spokes the selector names. The platform profile stays. |

A build that needs a new kernel says so on the package. A CNI that installs a kernel module is an OS wave, then a CNI wave. The wave is created as speed 3, so the drain is visible before you start.

kube-proxy and CoreDNS are packages. They are not fields inside the Kubernetes version. The node image carries the kernel, the OS userspace, and the container runtime. It does not carry the CNI, CSI, CCM, or Prometheus binaries the cluster runs. Those are pods. Which of today's products already ship those as separate artifacts, and which print one version over several of them, is [What you track](versioning.md#what-you-track).

## One wave, one component

A wave is the only way a version changes.

| Field | Meaning |
| --- | --- |
| `component` | One row from the table above. |
| `from`, `to` | The version leaving and the version entering. |
| `rings` | Ordered. The next ring waits for the gate. |
| `concurrency` | Clusters in the current ring that move at once. A speed 3 wave also carries `maxSurge` and `maxUnavailable` inside each cluster. |
| `gate` | Soak, max failed clusters, and that component's health contract. |
| `stop` | A failed health contract stops the ring. Later rings stay put. |

A Kubernetes minor is a queue of waves, one component each:

1. Platform apps whose charts must understand the next APIs. Charts that already support the minor stay on their pin.
2. Control plane. Workers keep their kubelet.
3. CCM, when the new apiserver would leave it more than one minor behind. Otherwise it stays.
4. Kubelet. This is the node roll.
5. kube-proxy, if you want it current. It can wait on its own wave while it remains inside its window.

CSI and CNI stay unless their package says they will not run on the new minor. The hub lists those clusters as blocked, with the package name, before the control-plane wave starts.

```yaml
apiVersion: platform.example/v1
kind: Wave
metadata:
  name: ebs-csi-1.41.0
spec:
  component: csi
  from: ebs-csi-1.40.0
  to: ebs-csi-1.41.0
  rings: [canary, early, broad]
  concurrency:
    clusters: 25
  gate:
    soak: 2h
    maxFailed: 0
    health: [volume-create, existing-mount]
```

Rollback writes `to` back to `from` and uses the same rings. There is one code path.

## Profiles, rings, clusters

**Profile.** The desired version of every platform component. Prod points at one profile. A wave edits one pin, or publishes the next profile and moves a ring onto it. A cluster carries its own copy of a version only as an override, and an override shows up as drift.

**Ring.** An ordered set of clusters that share a profile. Canary, early, broad, late. Membership is a label. A cluster that joins a ring converges to that profile.

**Cluster.** Desired from the ring, actual from the spoke. You open a cluster from a bad cell. You do not start from the cluster list.

Version numbers in the profile and the grid are one example, chosen so the lag matches [Compatibility](compatibility.md). They are not a release to install.

```yaml
apiVersion: platform.example/v1
kind: Profile
metadata:
  name: prod
spec:
  components:
    os: "2026.11.4"
    containerd: "2.1.3"
    kubernetes: "1.36.3"
    kubelet: "1.35.8"       # one minor behind; legal. The kubelet wave has not reached this ring.
    kube-proxy: "1.36.3"
    coredns: "1.12.0"
    cni: cilium-1.17.2
    csi: ebs-csi-1.40.0
    ccm: aws-ccm-1.36.1     # within one minor of the apiserver
    flux: "2.10.1"
    prometheus: "82.13.6"
```

Rings order clusters. Waves order one component. A wave does not reorder the other columns.

## Hub and spokes

The hub stores profiles, rings, waves, and packages. Git is the source of truth. Hub controllers are clients of that Git. A hand edit on a spoke is drift, and the next reconcile restores the profile. Same rule as [Concepts](concepts.md): write the change where the controller reads.

Spokes pull.

```
Git
 └─ hub: profiles, rings, waves
      └─ one OCI artifact per profile
           └─ spoke Flux pulls, applies, reports
                └─ hub writes status
                     └─ fleet grid
```

1. Git changes a profile or a wave.
2. The hub publishes one OCI artifact per profile. A cluster with an override gets its own artifact. The other 999 do not.
3. Flux on the spoke applies the artifact for its ring.
4. The spoke reports, per component, desired version, actual version, health, and the last transition.
5. The hub stores that on the cluster object. The dashboard reads the hub. Painting the home page does not open a thousand apiservers.

A push loop from the hub is a backfill for a spoke that cannot pull yet. Steady state is pull. At a thousand clusters, fan-out from one apiserver is the outage.

The hub upgrades as a wave whose ring contains the hub. Spoke profiles do not follow the hub version. That wave does not enqueue spoke waves.

The hub copy and the spoke copy are different binaries. A newer Flux, Prometheus, or Thanos on the hub does not patch the spoke, and a CVE fixed only in the hub image is still on the spoke until that spoke's wave. Flux's one-minor rule applies inside one cluster. The hub and a spoke do not share an apiserver, so the hub may sit further ahead than that, inside the window [Compatibility](compatibility.md) records for the pair. The grid shows the spoke's version. The hub has its own row.

### Apps from the hub to every spoke

Two planes, one mechanism.

| Plane | Who writes Git | What they can pin | Where it lands |
| --- | --- | --- | --- |
| Platform | Platform team | Every component in the profile | Every spoke in the rings the wave names |
| Apps | Platform team, or a workspace owner | Charts they are allowed to publish | Spokes matching a selector: a ring, a label, or all |

Prometheus is a column on the profile. A new chart is a wave on that column. It reaches every spoke in the named rings. You declare it once.

A catalog app or a tenant app uses the same wave, the same health contract, and the same status. It is not a pin on the platform profile, so a tenant release cannot move Cilium. On the app's own screen it is a column. On the platform grid it stays off, so the home page remains the platform components.

Flux on the spoke is a platform component. The wave that upgrades Flux has a boring requirement: the Flux that is running has to be able to apply the next Flux. The package states that skew. Canary it on one spoke before the ring.

## The dashboard

Three views. The home view is a grid, not a list of clusters.

### Fleet

Rows are rings. Columns are components. A converged cell is one version. A cell that is not one version expands to four counts: on the profile version, in the current wave, drifting, degraded.

Rings point at a profile revision, so two rings can differ by the pins a wave has promoted. This fleet is mid CSI wave. Canary is on the next profile: CSI 1.41.0, kubelet already at 1.36.3. Early is on that kubelet pin and is rolling CSI. Broad is still on the profile above: CSI 1.40.0, kubelet 1.35.8, one minor behind the apiserver and inside the window. That kubelet gap is a wave you have not promoted, not a second wave running. OS, Kubernetes, CNI, CCM, and Prometheus are one version across all 955 clusters.

| Ring | Clusters | os | kubernetes | kubelet | cni | csi | ccm | prometheus |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| canary | 5 | 2026.11.4 | 1.36.3 | 1.36.3 | cilium 1.17.2 | ebs 1.41.0 | aws-ccm 1.36.1 | 82.13.6 |
| early | 50 | 2026.11.4 | 1.36.3 | 1.36.3 | cilium 1.17.2 | ebs 1.40.0 → 1.41.0 | aws-ccm 1.36.1 | 82.13.6 |
| broad | 900 | 2026.11.4 | 1.36.3 | 1.35.8 | cilium 1.17.2 | ebs 1.40.0 | aws-ccm 1.36.1 | 82.13.6 |

A healthy fleet is one version per column. A wave in flight is two versions in that column, split by ring. Three versions in one column is the smell [Upgrade at scale](scale.md) warns about. The cell shows the version string. A green mark with no version is how a fleet collects three CSI builds.

Cluster names appear when you open a cell.

### Wave

The work view. One component, the rings, the gate, how many clusters have converged, which health check failed, rollback. The wave shows its speed. Speed 2 estimates minutes per batch. Speed 3 estimates a drain per node, and a second speed 3 wave on those clusters waits.

### Cluster

One row per component: desired, actual, health, the wave that last changed it, the event if health is false. This answers why one cluster is red. It is the wrong first page at a thousand clusters.

### Answers that stay on one screen

- Which component is mid-roll, and which ring it has reached.
- Whether prod is still on the previous profile.
- How many clusters are degraded for CSI, and that the cause is the wave.
- Whether a kubelet lag is inside skew.
- Whether the next artifact's digest is in the site mirror. A missing digest leaves the wave pending. The cell says `artifact missing`.

## A platform engineer's day

The job is publish the artifact, open a wave, watch the gate. Every component is the same object.

**CSI CVE.** `ebs-csi` 1.41.0 is in the mirror. Open a wave from 1.40.0 to 1.41.0. Canary is five clusters. The contract creates a volume and checks an existing mount. The DaemonSet rolls. The gate is two hours and zero failures. Early starts on its own at the concurrency you set. Nodes stay up. Kubernetes stays. Prometheus stays. The canary finishes in minutes. The fleet finishes the same day, or the gate stops it.

**Kernel CVE.** A different wave. Component `os`, new image digest, same kubelet pin. Speed 3, `maxSurge: 1`, `maxUnavailable: 0`. The clock is drains and PodDisruptionBudgets, as in [Upgrade at scale](scale.md). Charts stay on their pins. The Kubernetes minor stays put.

**Kubernetes minor.** The hub runs the matrix first. A cluster whose CSI or CNI range excludes the target minor is blocked, with the package named, and the other clusters can proceed. Then the queue: apps that need the new APIs, the control plane, CCM if its one-minor window requires it, the kubelet, kube-proxy when you want it current. Prod's kubelet wave has its own soak. A bad API shows up while workers are still on the old kubelet.

The package row already says the speed. You do not keep a private map of which vendor baked the CNI into the image.

## What the controllers enforce

| Rule | If it is broken |
| --- | --- |
| A wave names one component. | The API rejects a wave that sets two version fields. |
| The artifact digest is in the mirror for every site the ring covers. | The wave stays pending. The cell reads `artifact missing`. |
| The package's Kubernetes range covers that cluster's control plane and kubelet. | Those clusters are listed and left out. The rest proceed. |
| A package marked host-level ships as speed 3. | The wave is speed 3, with the drain visible, before it starts. |
| Spoke drift reconciles to the profile. | The cell stays `drifting` until actual matches desired. A hand edit does not become the new desired. |
| `maxFailed` defaults to zero. | One degraded cluster stops the ring. |
| Two speed 3 waves do not overlap on one cluster. | The second waits. |
| Tenant Git cannot write platform pins. | That commit does not change the profile. |

Where a provider cannot split a component, the column still exists. A managed CSI that only moves with the control plane is a binding on the control-plane wave: the wave lists `also moves: csi`. The grid shows both cells change together. The coupling is on the screen. The preferred build is still the split, because that is what makes a Tuesday CVE a DaemonSet roll.

## Air gap

Same objects. The extra column is the mirror. For each version the next wave wants, the digest is present or it is not. Stage digests, then open the wave. The spoke does not learn versions from the internet. Artifacts first, as in [Concepts](concepts.md).

## Where today's products already do a piece

"Closest" means that mechanism exists. The rest of the row is often missing. Details are on the platform pages, [CVE scope](cve-scope.md), and [Platform apps](platform-apps.md).

| Property | Closest today | Still missing |
| --- | --- | --- |
| CSI and CNI on their own version, speed 2 | EKS `update-addon` | GKE, AKS, and OpenShift payload CNI move with the platform. EKS Anywhere Cilium moves with the machines. NKP bundled add-ons move only to the build in that NKP release. |
| OS image, Kubernetes minor unchanged | AKS `--node-image-only`, an EKS AMI at the same version, RKE2 through the OS package manager, a newer GKE `-gke.N` inside the minor | An OpenShift z-stream moves RHCOS and the kubelet together. An NKP CAPX image is built per Kubernetes version, so the supported roll moves both. |
| Control plane, workers unchanged | GKE `--master`, EKS `update-cluster-version`, AKS `--control-plane-only`, OpenShift with the worker pool paused | On NKP CAPX and EKS Anywhere the new control-plane machine is already the new image. |
| Apps from a hub, versioned apart from Kubernetes | Kommander `AppDeployment`, Rancher Fleet `GitRepo`, ACM policies | NKP's default platform set bumps together when the platform moves. OpenShift monitoring is in the payload. GKE and AKS collectors often move with the node. |
| Rings, a gate, and stop-the-rest | AKS Fleet Manager update runs | The run types are Kubernetes and the node image. A CSI chart or Prometheus is not an update run. |
| One grid of every component | — | Each product exposes a different object. The inventory in [Upgrade at scale](scale.md) is something you assemble. Here it is status the spoke already reported. |

EKS has the add-on split. AKS Fleet has rings for Kubernetes and the node image. Kommander and Rancher Fleet deliver apps from a hub. The ideal is those as one wave object and one grid, covering the OS, the control plane, the kubelet, CSI, CNI, CCM, and Prometheus.

## Limits

A node roll still drains a node at a time. A thousand clusters with tight PodDisruptionBudgets still take days for an OS wave. Concurrency and surge are the same knobs as [Upgrade at scale](scale.md), with the same ceilings.

Pod CIDR, service CIDR, and the identity of the CNI plugin stay create-time choices. Speed 4, as in [Concepts](concepts.md). Out of band covers versions of components the cluster already runs.

The cell shows the version the spoke reported. A cell that only means "the agent is up" recreates the inventory problem this grid exists to end.
