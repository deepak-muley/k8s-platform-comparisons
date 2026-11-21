# OpenShift Container Platform

Self-managed OpenShift 4. The cluster is a set of operators reconciled by the Cluster Version Operator from a single release payload image. You do not upgrade kube-apiserver, etcd, and the OVN-Kubernetes DaemonSet as separate projects. You move the payload, and those operators move with it.

ROSA and ARO use the same payload and the same `ClusterVersion` object. The difference is who is allowed to change it. On ROSA with hosted control planes, Red Hat rolls the control plane and you roll the machine pools. The procedures below are for self-managed OCP, with a note where the actor changes.

Checked against the 4.18 and 4.19 disconnected-install and update docs.

OpenShift's Machine API is the ancestor of Cluster API, but a payload upgrade does not replace the VM. The Machine Config Operator rebases RHCOS on the existing node and reboots it. The kubelet and the OS commit move together; there is no OS-only image and no Kubernetes-only boot. Local disks and the etcd data directory survive that reboot. A VM is replaced when the machine spec changes (size, failure), not because the version changed. The CAPX behavior this is easy to confuse it with is in [Cluster API](../capi.md).

## Core vs add-on

| | What | Object you touch | Moves when |
| --- | --- | --- | --- |
| Core, payload | kube-apiserver, etcd, kube-controller-manager, kube-scheduler, machine-config, network (OVN-Kubernetes), ingress controller operator, image registry, authentication, console, monitoring, Cluster Storage Operator, CSI snapshot controller | `ClusterOperator` (`oc get co`). Desired version is `clusterversion/version`. | `oc adm upgrade` |
| Core, node | RHCOS and the kubelet, rendered from MachineConfigs | `MachineConfig`, `MachineConfigPool` (`master`, `worker`) | The same upgrade, after the operators. A paused pool waits. |
| Platform CSI for the cloud | AWS EBS, GCP PD, Azure Disk, vSphere, and similar Red Hat operators | A `Subscription` in a namespace such as `openshift-cluster-csi-drivers`, fed by the release's operator index | The catalog index that ships with the minor. Plan on it moving with the minor, not on an arbitrary upstream CSI tag. |
| Add-on | Anything else you subscribed to: logging, GitOps, Service Mesh, a third-party CSI, a partner operator | `Subscription`, `ClusterServiceVersion`, your Helm release | When you approve a new CSV, or when the approval is `Automatic` and the catalog has a newer one |

OVN-Kubernetes is the cluster network on current 4.x. `networking.networkType` in `install-config.yaml` is a create-time choice. Replacing it with Cilium or Calico as the cluster CNI is a new cluster.

## Where configuration lives

| Change | API | Speed |
| --- | --- | --- |
| Cluster version | `oc adm upgrade` or `oc patch clusterversion/version` | 3 for nodes, after operators roll |
| CNI behavior (MTU, IPsec, routing) | `oc edit network.operator.openshift.io cluster` | 3. The network operator writes a MachineConfig and the pool rolls. |
| Ingress replicas, certs, node placement | `oc edit ingresscontroller/default -n openshift-ingress-operator` | 1 or 2. Replica and route changes roll router pods, not nodes. |
| Node kernel arg, sysctl, chrony, container runtime | A `MachineConfig` labeled `machineconfiguration.openshift.io/role: worker` (or `master`) | 3. Every node in that pool reboots, in batches of `maxUnavailable`. |
| A StorageClass | `StorageClass` | 1 |
| An optional operator's config | The operator's CR (`ocs`, `odf`, a CSI `DriverConfig`, …) | 2, unless that CR writes a MachineConfig |

Hand-editing a pod the Cluster Version Operator owns is reverted. Change the operator CR or the MachineConfig.

## Update channels

A channel is not an install-time flag. The installer leaves the cluster on `stable-4.y` for the minor you installed. After that, the channel is `ClusterVersion.spec.channel`, and it only controls which releases Cincinnati will recommend. It does not change how a release is applied. Every recommended release is still one payload: apiserver, etcd, OVN-Kubernetes, the Machine Config Operator, and the RHCOS boot image.

The number in the channel is the minor you are willing to land on. `stable-4.18` while you are on 4.18 offers z-streams only. `stable-4.19` offers the path from 4.18 to 4.19. The cluster is only offered its current minor, the next minor, or the next EUS minor. A z-stream does not need a channel change.

```bash
oc get clusterversion version -o jsonpath='{.spec.channel}{"\n"}'
oc adm upgrade          # what this channel recommends right now
oc adm upgrade channel stable-4.19
```

| Channel | What lands in it | Support | Use it for |
| --- | --- | --- | --- |
| `stable-4.y` | Default on a new cluster. A release arrives here after it has been in `fast-4.y` long enough for Red Hat to look for regressions. Z-streams of the latest minor are often a week or two behind fast. The first z-stream of a new minor is often 45–90 days behind fast. | Fully supported. Same errata as fast, later. | Production that can wait. |
| `fast-4.y` | The release as soon as Red Hat marks it GA and publishes the errata. | Fully supported. A regression is handled the same as on stable. The only difference from stable is the delay. | You want a CVE fix the week it is published, and you accept the shorter soak. |
| `eus-4.y` | Even minors only (4.18, 4.20, 4.22). A release is copied here at the same time it enters stable. | Same bits as stable. The channel exists so an EUS-to-EUS control-plane update is a recommended path. | Staying on an even minor, then moving control plane to the next even minor while workers are still paused on the previous one. |
| `candidate-4.y` | Builds as soon as they exist, before GA. Features can disappear before GA. A candidate build may have no path forward to a GA release. | Not supported. | A cluster you are willing to throw away. |
| empty (`oc adm upgrade channel`) | No recommendations. | You then pass `--to-image` yourself. | A disconnected cluster with no in-site OpenShift Update Service. |

Switching `stable-4.y` to `fast-4.y` is supported. Switching back is supported. Switching to `candidate-4.y` is allowed and can put you on a build you cannot leave except by recreating the cluster. Do not take a version number that was never in the channel. If 4.19.2 was skipped, the update to 4.19.2 is not a supported path even though 4.19.1 and 4.19.3 exist.

A CVE in the payload (kubelet, apiserver, OVN, RHCOS) is a z-stream on this channel. Fast gets it first. Stable gets the same z-stream after the promotion delay, including when the z-stream is the CVE fix. There is no separate "security only" channel.

What that z-stream cannot do is move one of those components and leave the others. Pausing `mcp/worker` delays the worker reboot. It does not give the workers a new kubelet on the old RHCOS. Masters still reboot. Unpausing applies the same payload, OS and kubelet together. The scenarios for every product are in [CVE scope](../cve-scope.md).

## Connected install (IPI)

IPI asks the platform API to create the machines. You need DNS for `api.<name>.<base>` and `*.apps.<name>.<base>` before you start, plus the platform credentials and a Red Hat pull secret.

```bash
openshift-install create install-config --dir=ocp
# edit ocp/install-config.yaml
openshift-install create cluster --dir=ocp --log-level=info
export KUBECONFIG=ocp/auth/kubeconfig
oc get clusterversion
oc get co
oc get mcp
```

The network stanza that matters:

```yaml
apiVersion: v1
baseDomain: example.com
metadata:
  name: prod
networking:
  networkType: OVNKubernetes
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  serviceNetwork:
    - 172.30.0.0/16
  machineNetwork:
    - cidr: 10.0.0.0/16
controlPlane:
  name: master
  replicas: 3
compute:
  - name: worker
    replicas: 3
platform:
  aws:            # or vsphere, azure, gcp, nutanix, none
    region: us-east-1
pullSecret: |
  {"auths":{...}}
sshKey: |
  ssh-ed25519 AAAA...
```

`platform.none` is UPI: you create the machines and the load balancer, the installer only generates ignition. Use it when the platform has no IPI provider you trust. Agent-based install is the better default for that case on current releases. It builds an ISO, the agent boots, and it drives the install without a live bootstrap VM you have to host.

Fields in that file, and what you get if you leave them at the generated value:

| Field | Generated value | What it does | After install |
| --- | --- | --- | --- |
| `baseDomain` | You type it. No default. | Parent DNS zone. The installer asks for `api.<name>.<baseDomain>` and `*.apps.<name>.<baseDomain>`. | Does not move. |
| `metadata.name` | You type it. | Cluster name. Becomes the left-most label of those DNS names. | Does not move. |
| `networking.networkType` | `OVNKubernetes` | The only cluster network for a current install. | Does not move. Another CNI is a new cluster. |
| `networking.clusterNetwork` | `10.128.0.0/14`, `hostPrefix: 23` | Pod CIDR, and how many pod IPs each node gets (`/23` out of that `/14`). | Does not move. |
| `networking.serviceNetwork` | `172.30.0.0/16` | Service CIDR. | Does not move. |
| `networking.machineNetwork` | The installer's reading of the platform subnet, when it can see one. | Node network. Used for the machine CIDR, not for pods. | Does not move. |
| `controlPlane.replicas` | `3` | Master count. `1` is single-node OpenShift. | Scale only by the documented procedure for that topology. |
| `compute.replicas` | `3` | Worker count. `0` is a three-node compact cluster: workloads are schedulable on the masters. | Add a MachineSet, or scale the existing one. |
| `compute.platform` / `controlPlane.platform` | The platform default size (on AWS, the installer picks an instance type for the region). | Machine type, disk, zones. Set per role when masters and workers differ. | A new MachineSet. The existing machines are not resized in place. |
| `platform.<name>` | You pick the cloud at the prompt: `aws`, `azure`, `gcp`, `vsphere`, `nutanix`, `none`, and others the release's installer knows. | Where the machines come from. `none` means you boot them. | Does not move. |
| `pullSecret` | You paste it. Required. | Auth to `registry.redhat.io` / `quay.io`. | `oc set data secret/pull-secret -n openshift-config`. |
| `sshKey` | You paste it, or leave it empty. | The key that can SSH to RHCOS as `core`. There is no password. | A MachineConfig. Workers reboot into it. |
| `publish` | `External` | `External` gives the API and ingress a public address. `Internal` keeps them on the private network. | Effectively create-time. Changing it is a reinstall-scale project. |
| `fips` | `false` | FIPS mode for the cluster. | Create-time. |
| `credentialsMode` | The platform default when omitted. | `Mint` lets the installer create cloud credentials. `Passthrough` uses the ones you supplied. `Manual` means you create the credential secrets yourself and no minting happens. Set `Manual` for accounts that cannot create IAM users, and for most disconnected installs. | Painful to change. Decide at install. |
| `imageContentSources` | Absent | Registry mirror rewrite. Required for a disconnected install. | `ImageDigestMirrorSet` / `ImageContentSourcePolicy` after install. |
| `additionalTrustBundle` | Absent | PEM CA bundle added to the node trust store. The registry CA and the proxy CA go here. | A MachineConfig. Nodes reboot. |
| `proxy` | Absent | `httpProxy`, `httpsProxy`, `noProxy`. | `proxy/cluster`. Many proxy changes roll nodes. |
| `featureSet` | Absent, which is `Default` | `TechPreviewNoUpgrade` turns on tech-preview features and the cluster cannot return to `Default`. Some updates are then blocked. | Treat as permanent. |
| channel | Not in this file. | Set after install. A new cluster is on `stable-4.<minor of the release image>`. | `oc adm upgrade channel`. |

There is no install-config field that subscribes you to fast, and none that says "patch RHCOS but not the kubelet." Both of those are day-2, and the second one does not exist. See the channel table above.

After install:

```bash
oc get nodes
oc get storageclass
oc get network.operator.openshift.io cluster -o yaml
oc get pods -n openshift-ovn-kubernetes
```

## Air-gapped install

Mirror first, on a host that can reach `registry.redhat.io` and your registry.

`imageset-config.yaml`:

```yaml
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v2alpha1
mirror:
  platform:
    channels:
      - name: stable-4.19
        minVersion: "4.19.0"
        maxVersion: "4.19.6"
    graph: true
  operators:
    - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.19
      packages:
        - name: aws-ebs-csi-driver-operator
        - name: odf-operator
        - name: openshift-gitops-operator
  additionalImages:
    - name: registry.redhat.io/ubi9/ubi:latest
```

```bash
oc-mirror --config imageset-config.yaml docker://registry.site.example:5000 --v2
```

Take the results directory to the dark site. It contains the `ImageDigestMirrorSet` and the release signature ConfigMaps. Apply both to a cluster that already exists. For a new cluster, fold the mirror into `install-config.yaml`:

```yaml
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  ...registry CA...
  -----END CERTIFICATE-----
imageContentSources:
  - mirrors:
      - registry.site.example:5000/openshift/release
    source: quay.io/openshift-release-dev/ocp-release
  - mirrors:
      - registry.site.example:5000/openshift/release-images
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```

Use the sources `oc-mirror` printed. Do not invent them. The Agent-based installer is the usual way to boot dark nodes: generate the ISO from the mirrored release, boot it, and let the agent pull only from the mirror.

Also mirror the graph if you want `oc adm upgrade` to show a list later. That is `graph: true` plus OpenShift Update Service inside the site. See [Air gap](../airgap.md).

## Day-2 config

Kernel arg on workers, which reboots them:

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 99-worker-kargs
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  kernelArguments:
    - nosmt
```

```bash
oc apply -f 99-worker-kargs.yaml
oc get mcp worker -w
# UPDATING=True until every worker has rebooted into the new rendered config
```

Throttle:

```bash
oc patch mcp worker --type merge -p '{"spec":{"maxUnavailable":1}}'
```

Freeze the workers while masters move:

```bash
oc patch mcp worker --type merge -p '{"spec":{"paused":true}}'
# ...upgrade or a master-only MachineConfig...
oc patch mcp worker --type merge -p '{"spec":{"paused":false}}'
```

A new StorageClass does not roll anything:

```bash
oc apply -f - <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF
```

Remove the default annotation from the previous class in the same change. Two defaults make dynamic provisioning pick one at random.

## Upgrade the cluster

Preconditions: etcd backup you have actually restored once, every `ClusterOperator` available, no paused pool you forgot about, every installed operator able to move to the target minor. A z-stream on the current channel is the CVE path for everything in the payload. It does not let you take the RHCOS fix without the kubelet, or the OVN fix without both. Pausing `mcp/worker` only delays the worker half. Component-by-component options on the other platforms are in [CVE scope](../cve-scope.md).

```bash
oc adm upgrade
# lists the Cincinnati recommendations
oc get clusterversion
oc adm upgrade --to=4.19.6
oc get clusteroperators -w
oc get mcp -w
```

Order inside that one command:

1. CVO pulls the payload and rolls cluster operators.
2. Machine Config Operator rolls `master` (one at a time; the pool's `maxUnavailable` for masters stays 1).
3. Then `worker`, `maxUnavailable` at a time.

Watch `oc get co` before you watch nodes. A degraded `network` or `etcd` operator is the upgrade. The node roll has not started yet, and unpausing workers will not fix it.

### Disconnected upgrade

With OpenShift Update Service in-cluster, point CVO at it once:

```bash
POLICY_ENGINE_GRAPH_URI="$(oc -n openshift-update-service get updateservice service \
  -o jsonpath='{.status.policyEngineURI}/api/upgrades_info/v1/graph{"\n"}')"
oc patch clusterversion version --type merge \
  -p "{\"spec\":{\"upstream\":\"${POLICY_ENGINE_GRAPH_URI}\"}}"
```

The CVO has to trust that route's CA. A self-signed default ingress cert is a common reason the graph stays empty. Put the CA in the proxy trust or replace the ingress cert.

Without the update service:

```bash
oc apply -f release-signatures/
oc adm upgrade --allow-explicit-upgrade \
  --to-image registry.site.example:5000/openshift/release@sha256:DIGEST
```

Update every OLM operator to a CSV that exists for the target minor *before* this. When the catalog index flips from `v4.18` to `v4.19` mid-upgrade, an operator with no path in the new index goes `UpgradePending` and stays there.

## CNI

You operate OVN-Kubernetes through the network operator, not by editing DaemonSets.

```bash
oc get network.operator.openshift.io cluster -o yaml
oc edit network.operator.openshift.io cluster
```

Fields operators actually change:

- `defaultNetwork.ovnKubernetesConfig.mtu`
- `defaultNetwork.ovnKubernetesConfig.gatewayConfig` routing via host or shared gateway
- IPsec / encryption config the release supports
- `migration` only when Red Hat documents a migration for that pair of versions

Expect a MachineConfig and a full node roll. Pods keep their IPs. The cluster network CIDR does not change here.

Verify after the pool is done:

```bash
oc get pods -n openshift-ovn-kubernetes
oc get co network
oc debug node/<node> -- chroot /host ovn-appctl --help >/dev/null
```

There is no supported "upgrade just the CNI this afternoon" command. A z-stream that contains an OVN fix is the fast path, and it is still a cluster upgrade.

## CSI

Find what you have:

```bash
oc get csidriver
oc get subscription -A
oc get pods -n openshift-cluster-csi-drivers
oc get storageclass
```

Red Hat's cloud CSI operators follow the release catalog. To move them you move the catalog (a cluster upgrade, or a new index you mirrored) and then approve the install plan:

```bash
oc get installplan -n openshift-cluster-csi-drivers
oc patch installplan <name> -n openshift-cluster-csi-drivers \
  --type merge -p '{"spec":{"approved":true}}'
oc get csv -n openshift-cluster-csi-drivers -w
```

That rolls the CSI controller Deployment and the node DaemonSet. Volumes that are already mounted stay mounted. New mounts use the new node plugin once the DaemonSet pod on that node is ready. Do this before a Kubernetes minor when the vendor says the current CSI CSV is not valid on the next minor.

A third-party CSI (ODF/Rook, Portworx, Trident, Dell) is an operator you subscribed to or a Helm release. Upgrade that subscription or chart on its own schedule. Test, on a non-default StorageClass:

- create a PVC and a pod that mounts it
- expand the PVC (`allowVolumeExpansion: true`)
- snapshot and restore if you installed the snapshot CRDs
- delete the pod, confirm the volume reattaches on a different node
- delete the PVC, confirm the backend volume is released according to `reclaimPolicy`

In-tree volume plugins are gone from current OpenShift. If a manifest still says `provisioner: kubernetes.io/aws-ebs`, it will not provision. The provisioner string is the CSI one (`ebs.csi.aws.com`, `pd.csi.storage.gke.io`, `disk.csi.azure.com`, `csi.vsphere.vmware.com`, `csi.nutanix.com`).

## Rollback

A minor-version payload does not roll back in place. The supported recovery for a failed minor is the etcd backup you took before `oc adm upgrade`, restored with `oc adm release` / the cluster-etcd-operator procedure for that minor, or a reinstall. A z-stream that has not yet rolled every pool can be paused (`oc patch mcp worker --type merge -p '{"spec":{"paused":true}}'`) so the remaining nodes stay on the old rendered config while you fix the operator.

Take the backup immediately before the upgrade, from a healthy etcd, and keep it off the cluster.

## At scale

Advanced Cluster Management placements select clusters. A policy sets `spec.desiredUpdate` on `clusterversion/version`. Prod and nonprod should be different placements.

Inside each cluster, pause `mcp/worker` so masters and operators finish before any workload node reboots. One-node canary pools are a second `MachineConfigPool` with a node selector, not a special ACM feature.

Hosted control planes (HyperShift) move the control plane as pods on a management cluster. Worker machine pools still roll when their version changes. Size the management cluster for control-plane pod count.

Details and the pause/unpause sequence: [Upgrade at scale](../scale.md).

## Traps

- Pausing a pool and forgetting it. Those nodes sit on the old kubelet past the supported skew, and the next upgrade looks unrelated.
- Mirroring the payload and not the operator index. Day-2 CSI install fails closed.
- Two default StorageClasses after a CSI operator install. The operator often creates one and marks it default.
- Editing `oc get ds -n openshift-ovn-kubernetes` to change MTU. The network operator reverts it, or the next upgrade does.
- `--to-image` without the signature ConfigMap. CVO refuses the digest.
- ROSA: `oc adm upgrade` on a cluster where Red Hat owns the control plane schedule will fight the managed upgrade. Use the ROSA upgrade controls for the control plane, and machine pools for the nodes.
