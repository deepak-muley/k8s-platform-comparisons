# Amazon EKS Anywhere

EKS Anywhere is the EKS distro you run on your own machines: vSphere, bare metal through Tinkerbell, Nutanix, Snow, or Docker for a lab. AWS does not run the control plane. A management cluster (or a standalone cluster) is the thing you upgrade with the CLI. Workload clusters are `Cluster` objects on that management cluster, so GitOps and Terraform can upgrade them without the CLI on a laptop.

There is no automatic upgrade. Skipping an EKS Anywhere minor is unsupported. Management components may be at most one EKS Anywhere minor ahead of the cluster components.

Under the CLI this is Cluster API: CAPV on vSphere, CAPX on Nutanix, Tinkerbell on bare metal, Snow on AWS Snow. A `kubernetesVersion` change replaces machines. It does not `kubeadm upgrade` the VM you have. The OS image EKS Anywhere publishes is built for one Kubernetes version, so leaving the old template in place is not a supported way to move only the kubelet. A new template or `osImageURL` at the same `kubernetesVersion` is the OS-only roll, and it still reprovisions. Bare metal needs a spare host for `maxSurge`. Same rules as NKP on CAPX, written up in [Cluster API](../capi.md).

## Core vs add-on

| | What | Where it is set | Moves when | Speed |
| --- | --- | --- | --- | --- |
| Core | etcd, API server, kubelet, the EKS Anywhere admin controllers | `spec.kubernetesVersion` and `spec.eksaVersion` on `anywhere.eks.amazonaws.com/v1alpha1 Cluster` | `eksctl anywhere upgrade cluster` rolls machines. Control plane machines first, then workers. | 3 |
| Platform core | Cilium, unless you picked kindnet | `spec.clusterNetwork.cniConfig.cilium: {}` | With the cluster upgrade. You do not helm-upgrade Cilium on the side and stay supported. | 3, because the node rolls as part of the same upgrade. The CNI DaemonSet also restarts. |
| Not in the product since v0.17 | vSphere CSI, and every other CSI | Your Helm release | Whenever you want | 2 |
| Curated packages | Harbor, Prometheus, and the other packages in the EKS Anywhere package catalog | Package objects the package controller reconciles | A separate package upgrade, after the images are in the mirror if you are air-gapped | 2 |
| Your apps | Helm, GitOps | Your repo | You | 2 |

`cniConfig` is immutable after create. Cilium and kindnet are the two choices. Kindnet is for non-production. One plugin, no swap later.

CSI being absent is the useful part. A storage vendor chart moves on a storage-vendor schedule, including the afternoon of a CVE, without a new VM image. You own the compatibility matrix between that chart and the Kubernetes minor.

## Install, connected

Admin machine: Docker, `eksctl anywhere`, and credentials for the provider.

```bash
eksctl anywhere generate clusterconfig "${CLUSTER}" --provider vsphere > cluster.yaml
# or: --provider nutanix, --provider baremetal, --provider snow
```

Set the CNI explicitly so the next person does not have to guess:

```yaml
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: Cluster
metadata:
  name: prod
spec:
  kubernetesVersion: "1.32"
  clusterNetwork:
    pods:
      cidrBlocks:
        - 192.168.0.0/16
    services:
      cidrBlocks:
        - 10.96.0.0/12
    cniConfig:
      cilium: {}
  controlPlaneConfiguration:
    count: 3
  workerNodeGroupConfigurations:
    - count: 3
      name: md-0
```

Provider blocks (vSphere datacenter, Nutanix Prism endpoint, bare-metal MAC and BMC) go in the same file, in the provider-specific kinds the `generate` command wrote. Fill those in from the environment. Do not commit passwords. The CLI can read them from the environment and from files on the admin machine.

```bash
eksctl anywhere create cluster -f cluster.yaml
export KUBECONFIG="${PWD}/${CLUSTER}/${CLUSTER}-eks-a-cluster.kubeconfig"
kubectl get nodes
kubectl get pods -n kube-system -l app.kubernetes.io/name=cilium
kubectl get csidriver
```

`csidriver` is empty until you install one. On vSphere that is the vSphere CSI Helm chart, pinned to a version that supports this Kubernetes minor. On Nutanix it is the Nutanix CSI chart. EKS Anywhere will not install either for you.

A management cluster is a cluster you create with the management-cluster flow, and then you create workload clusters *on* it (`eksctl anywhere create cluster -f workload.yaml --kubeconfig management.kubeconfig`, or by applying the workload spec). Standalone is simpler and does not give you a place to park a fleet. Use a management cluster once you have more than one workload cluster.

## Air-gapped install

The admin machine needs internet once, and on the order of 80 GB free. The download pulls images for every provider and every supported Kubernetes version. Run it on amd64 to import amd64 images.

```bash
eksctl anywhere download artifacts
tar -xvf eks-anywhere-downloads.tar.gz
eksctl anywhere download images -o images.tar

export REGISTRY_MIRROR_URL=registry.site.example:443
eksctl anywhere import images \
  -i images.tar \
  -r "${REGISTRY_MIRROR_URL}" \
  --bundles ./eks-anywhere-downloads/bundle-release.yaml
```

`--insecure` on `import images` if the registry uses a private CA and you have not installed it on the admin machine yet. The cluster still needs the CA in `registryMirrorConfiguration`. A skip-verify import does not configure the nodes.

Add the mirror to the spec before `create cluster`:

```yaml
spec:
  registryMirrorConfiguration:
    endpoint: registry.site.example
    port: "443"
    caCertContent: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
    insecureSkipVerify: false
```

Curated packages are a second copy, from ECR into the same registry. They are not in `images.tar`.

Bare metal also needs the node OS and the Tinkerbell hook OS on an internal URL:

```yaml
spec:
  datacenterRef:
    kind: TinkerbellDatacenterConfig
    name: prod
  # on the Tinkerbell machine / template side:
  # osImageURL: http://files.site.example/ubuntu-eks-a.ova-or-raw
  # hookImagesURLPath: http://files.site.example/hook
```

If those URLs are wrong you get machines that power on and never become Kubernetes nodes. The cluster object looks like a CNI failure. The BMC console shows a machine with no OS.

After the mirror is populated, the admin machine and the cluster can lose internet. Upgrades need a new import first. See [Air gap](../airgap.md).

## Upgrade

Upgrade the CLI to the target EKS Anywhere minor before `upgrade plan`. The CLI is what knows the new component versions.

```bash
eksctl anywhere upgrade plan cluster -f cluster.yaml
```

Read the plan. It lists component versions that will change. Then set `spec.kubernetesVersion` and, when the EKS Anywhere minor is moving, `spec.eksaVersion`.

Throttle the roll before you start:

```yaml
spec:
  controlPlaneConfiguration:
    count: 3
    upgradeRolloutStrategy:
      rollingUpdate:
        maxSurge: 1
        maxUnavailable: 0
  workerNodeGroupConfigurations:
    - name: md-0
      count: 6
      upgradeRolloutStrategy:
        rollingUpdate:
          maxSurge: 1
          maxUnavailable: 0
```

```bash
eksctl anywhere upgrade cluster -f cluster.yaml
```

Control plane machines are replaced first, then workers. `maxUnavailable: 0` with `maxSurge: 1` means a new machine must be Ready before an old one is removed. You need spare capacity on vSphere, Nutanix, or a spare bare-metal host. Without spare capacity, surge stays at 0 in practice and the roll blocks.

Workload clusters can be upgraded without the CLI, by editing the same fields and applying the object to the management cluster with kubectl, Flux, or Terraform. The management cluster's controllers run the roll. Keep the management cluster's EKS Anywhere version within one minor of those workload clusters.

Air gap: repeat `download artifacts`, `download images`, and `import images` with the *new* CLI before `upgrade cluster`. On bare metal, point `osImageURL` and `hookImagesURLPath` at the new images first.

There is no phone-home if you do not. `upgrade plan` uses the local CLI and the cluster. It does not need the public Cincinnati-style graph.

### What you get by not skipping minors

`v0.23` to `v0.25` in one step is unsupported. `v0.23` to `v0.24` to `v0.25` is the path. Each step is a machine roll. Plan the calendar as one minor per change window, not as a catch-up Friday.

## CNI

Create:

```yaml
cniConfig:
  cilium: {}
```

or

```yaml
cniConfig:
  kindnetd: {}
```

After create, the field is fixed. A dry-run of a spec that changes `cilium` to `kindnetd` is not an upgrade plan. It is a new cluster and a workload migration.

Cilium's version moves when EKS Anywhere moves. If you need a Cilium fix that the current EKS Anywhere patch does not contain, the supported options are: wait for the EKS Anywhere build that contains it, or take ownership of CNI by rebuilding on a distro where Cilium is an add-on you helm-upgrade (RKE2 with `cni: none`, or NKP with the CNI `ClusterResourceSet` removed at create). Running a second Cilium next to the EKS Anywhere Cilium will fight over the CNI socket on the next node replacement.

Config that on other platforms is a Helm value (MTU, policy mode, Hubble) is not a supported free-form field on the EKS Anywhere `Cluster`. Check the spec reference for the minor you run before you promise a Hubble-only change "with no node roll". If the field does not exist, the change is a cluster upgrade or it is unsupported.

Verify:

```bash
kubectl get ds -A -l app.kubernetes.io/name=cilium
kubectl get nodes
kubectl get pods -o wide | head
```

A node that is `Ready` with an empty `kube-system` Cilium pod list is a node whose CNI never came up. New pods will stay in `ContainerCreating`.

## CSI

Install and upgrade like any other Helm chart. vSphere shape:

```bash
helm upgrade --install vsphere-csi vsphere-csi/vsphere-csi \
  --namespace vmware-system-csi --create-namespace \
  --version "${CHART_VERSION}" \
  -f values.yaml
```

`values.yaml` holds the vCenter endpoint and the credentials secret reference. Keep credentials out of the command line history.

Nutanix:

```bash
helm upgrade --install nutanix-csi ./nutanix-csi-storage \
  --namespace ntnx-system --create-namespace \
  --set prismCentralEndPoint="${PC}" \
  --set kubernetesClusterDeploymentType=bare-metal
```

Use the chart version the storage vendor lists for this Kubernetes minor. EKS Anywhere's release notes will not pin it for you.

Then a StorageClass, for example `provisioner: csi.vsphere.vmware.com` or `provisioner: csi.nutanix.com`, `volumeBindingMode: WaitForFirstConsumer`, `allowVolumeExpansion: true`.

Because a cluster upgrade replaces the VM, test CSI across that boundary on the canary cluster:

- a volume created before the upgrade still mounts on a new node
- a volume created on a new node mounts
- expansion works
- a snapshot restores, if you installed the snapshot controller (EKS Anywhere does not install one for you)

The snapshot CRDs and the external-snapshotter Deployment are yours as well. A CSI driver without them is fine until someone applies a `VolumeSnapshot`.

## At scale

One management cluster, many workload `Cluster` objects. The fan-out is whatever applies those objects.

Practical pipeline:

1. Import the new air-gap images once.
2. Upgrade the management cluster's components (`upgrade management-components` when you are moving only the management side, or `upgrade cluster` when the management cluster's own Kubernetes version moves).
3. Apply the new spec to one nonprod workload cluster. `maxSurge: 1`, `maxUnavailable: 0`.
4. Wait until every Machine is `Running` and the Kubernetes version matches.
5. Apply the next cluster.

Do not commit a Git change that bumps `kubernetesVersion` on every workload cluster in one pull request unless the pipeline serializes the apply. Flux will do exactly what the Git repo says, in parallel, and you will roll every cluster on a Friday.

`upgrade plan` on the canary is the pre-check. Read it. A plan that shows an unexpected component jump means the CLI and the spec are not the pair you think they are.

## Traps

- Changing `cniConfig` in place and treating the resulting error as a bug. It is immutable.
- Forgetting to import images with the same CLI version you will use to upgrade. Preflight fails looking for a bundle manifest that does not match.
- Bare-metal `osImageURL` still pointing at the previous OS. New machines boot the old image, or none.
- Expecting the vSphere CSI driver to be there because an older lab (pre-0.17) had it. It is not.
- `maxUnavailable` left at the default on a small cluster. You can drain the only node a single-replica workload could run on.
- Upgrading workload clusters to an EKS Anywhere minor the management cluster has not reached, or more than one minor behind. Stay inside the one-minor management skew.
- Putting the registry password in `cluster.yaml` and pushing the file. Use the admin machine's local secret store and an untracked overlay.
