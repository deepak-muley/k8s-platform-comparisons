# Nutanix Kubernetes Platform

NKP is a Cluster API distribution with a fleet layer on top. The CLI (`nkp`) creates a temporary bootstrap cluster, installs CAPI, builds the real cluster, and then deletes the bootstrap. Kommander is the management UI, the platform-app catalog, and the workspace model. It runs on the management cluster.

A management cluster creates managed clusters. You upgrade the management cluster first, because that is where the new CAPI controllers and ClusterClass live.

Commands below match the NKP 2.17 air-gap flow and the 2.17–2.19 upgrade guide. NKP 2.19 added a single per-cluster upgrade path for ClusterClass clusters (platform apps first, then Kubernetes, infrastructure, and add-ons) and `nkp upgrade platform-apps` for clusters that are not ClusterClass. The sequenced commands still describe what that wrapper does. Dry-run on the minor you are actually running.

## Core vs add-on

| | What | Object | Moves when |
| --- | --- | --- | --- |
| Kubernetes core | API server, etcd, scheduler, controller-manager, kubelet | CAPI `Cluster`, `Machine`, `KubeadmControlPlane`, `MachineDeployment`. On current Nutanix builds, an `NKPCluster`. | `nkp upgrade cluster <provider> --vm-image ...` replaces machines with a new image |
| CAPI controllers | The management plane that creates those machines | Deployments in `capi-system` and the provider namespace (`capx-system` on Nutanix) | `nkp upgrade capi-components`, or the management-cluster upgrade on Nutanix, which does this itself |
| Core add-ons | CNI, the CSI for that provider, cluster-autoscaler, Node Feature Discovery | `ClusterResourceSet` + `ConfigMap`, one set per cluster, named like `calico-cni-installation-<cluster>` or the Cilium equivalent | `nkp upgrade addons <provider> --cluster-name`. On Nutanix this is also inside `nkp upgrade cluster nutanix`. |
| Platform apps | The default Kommander set: monitoring, logging, Traefik, cert-manager, Flux, gatekeeper, and the rest of that release's platform list | `AppDeployment`, `HelmRelease` | Upgraded as one set, before Kubernetes. `nkp upgrade kommander` then `nkp upgrade workspace` through 2.18. On 2.19, `nkp upgrade platform-apps` (`nkp upgrade workspace` is deprecated). |
| Catalog apps | Harbor, Istio, and anything you enabled from a catalog collection | `AppDeployment` you created | Not in that set. Upgrade one app when the catalog collection has a new version. |
| Your apps | Anything you installed that Kommander does not own | Your Helm release or GitOps repo | You |

Core add-on customizations made by editing the `ClusterResourceSet` are not preserved. `nkp upgrade addons` rewrites those objects. Keep the customization as a patch you reapply from `nkp upgrade addons --dry-run -o yaml`, or stop using the bundled add-on and install your own (CNI removed at create time, CSI via Helm).

### Which CNI you will actually get

It depends on the provider and the minor.

- Current NKP on Nutanix infrastructure deploys Cilium. A healthy cluster has pods you can list with `-l app.kubernetes.io/name=cilium`.
- The upgrade guide's generated resources for AWS (and the same `nkp upgrade addons` flow for azure, vsphere, gcp, eks, preprovisioned) still show a Calico `ClusterResourceSet` (`calico-cni-installation-<cluster>`, `tigera-operator-<cluster>`).

Do not assume from the provider name. Generate the manifest:

```bash
nkp create cluster nutanix --cluster-name probe --dry-run -o yaml > cluster.yaml
grep -n -i -E 'cni|cilium|calico' cluster.yaml
```

## Connected install (management cluster on Nutanix)

Prerequisites that are not optional in production:

- Prism Central reachable from the bastion, with a user that can create VMs, images, and subnets.
- An NKP node image already uploaded to Prism. The image name is what the CLI looks up. Do not rename the image you downloaded with that NKP version.
- A free IP for the control-plane endpoint, and a range for `kubernetes-service-load-balancer-ip-range` (MetalLB-style services, including the Kommander UI).
- A storage container for the Nutanix CSI.
- DNS you control, NTP, and an SSH public key whose private key stays on the bastion.

```bash
export CLUSTER_NAME=nkp-mgmt
export NUTANIX_ENDPOINT=prism.site.example
export NKP_IMAGE=nkp-ubuntu-24.04-1.32.x-YYYYMMDDHHMMSS
export STORAGE_CONTAINER=default-container

nkp create cluster nutanix \
  --cluster-name "${CLUSTER_NAME}" \
  --endpoint "https://${NUTANIX_ENDPOINT}:9440" \
  --control-plane-prism-element-cluster "${NUTANIX_PE_CLUSTER}" \
  --control-plane-subnets "${SUBNET}" \
  --control-plane-endpoint-ip "${CONTROL_PLANE_VIP}" \
  --control-plane-replicas 3 \
  --control-plane-vm-image "${NKP_IMAGE}" \
  --worker-prism-element-cluster "${NUTANIX_PE_CLUSTER}" \
  --worker-subnets "${SUBNET}" \
  --worker-replicas 4 \
  --worker-memory 32 \
  --worker-vm-image "${NKP_IMAGE}" \
  --kubernetes-service-load-balancer-ip-range "${LB_IP_RANGE}" \
  --csi-storage-container "${STORAGE_CONTAINER}" \
  --ssh-username "${SSH_USERNAME}" \
  --ssh-public-key-file "${SSH_PUBLIC_KEY}" \
  --insecure \
  --self-managed

nkp get kubeconfig --cluster-name "${CLUSTER_NAME}" > "${CLUSTER_NAME}.conf"
export KUBECONFIG="${CLUSTER_NAME}.conf"
kubectl get nodes
kubectl get pods -A
kubectl get storageclass
```

`--self-managed` installs Kommander onto this cluster and makes it a management cluster. `--insecure` skips TLS verify to Prism. Use a real CA (`--additional-trust-bundle`) when Prism has an internal CA. The trust bundle flag has a 16 KB cloud-init limit.

What the command does, in order:

1. Starts a kind bootstrap cluster on the bastion (Docker must be running).
2. Installs CAPI and the Nutanix provider into it.
3. Creates the control-plane VMs and the worker VMs from `NKP_IMAGE`.
4. Pivots CAPI onto the new cluster and deletes the kind cluster.
5. Applies core add-ons (CNI, Nutanix CSI, autoscaler, NFD) as `ClusterResourceSet`s.
6. With `--self-managed`, installs Kommander and the default platform apps via Flux.

A workload cluster is the same command with `--kubeconfig` pointed at the management cluster and `--namespace` set to the workspace namespace, and without making it another management cluster.

### Pre-provisioned

The OS is already installed. NKP SSHes in as a user with passwordless sudo and lays Kubernetes down. Swap is off, the iSCSI daemon is on if you want Nutanix volumes, and the SSH user exists on every node.

```bash
nkp create cluster preprovisioned --help

nkp create cluster preprovisioned \
  --cluster-name "${CLUSTER_NAME}" \
  --control-plane-endpoint-ip "${CONTROL_PLANE_VIP}" \
  --pre-provisioned-inventory-file inventory.yaml \
  --ssh-private-key-file "${SSH_PRIVATE_KEY}"
```

Add `--airgapped` and the registry flags when the nodes cannot pull from the internet. Confirm the inventory flag against `--help` for the minor you are running. The file lists the control-plane and worker addresses NKP will SSH to.

Nutanix CSI is not always part of that path. Install it yourself:

```bash
# versions must match the NKP minor; take them from the support portal
kubectl kustomize external-snapshotter/client/config/crd | kubectl apply -f -
kubectl kustomize external-snapshotter/deploy/kubernetes/snapshot-controller | kubectl apply -f -

helm install nutanix-csi ./nutanix-csi-storage \
  --namespace ntnx-system --create-namespace \
  --set kubernetesClusterDeploymentType=bare-metal \
  --set createSecret=false \
  --set prismCentralEndPoint="${NUTANIX_ENDPOINT}" \
  --set pcUsername="${PC_USER}" \
  --set pcPassword="${PC_PASSWORD}"
```

Then a StorageClass with `provisioner: csi.nutanix.com`, `storageType: NutanixVolumes`, the Prism Element UUID, and the storage container. If `local-volume-provisioner` is still the default class, remove that annotation. This Helm release is yours. Upgrade it with `helm upgrade` on your own cadence. It is the fast CSI path on NKP.

### Bring a different CNI

Only at create, before nodes need networking:

```bash
nkp create cluster nutanix --cluster-name "${CLUSTER_NAME}" \
  <the rest of the flags> \
  --dry-run -o yaml > cluster.yaml
```

Delete whichever of these the file contains:

- `ClusterResourceSet` `calico-cni-installation-<cluster>` and the ConfigMaps `calico-cni-installation-<cluster>` and `tigera-operator-<cluster>`, or
- the `cni` block under `addons` in the cluster config, or
- the Cilium `ClusterResourceSet` the Nutanix provider generated

Leave the CSI, load balancer, and NFD add-ons. Apply the file. Install the replacement CNI immediately, before you expect pods to schedule. `PreprovisionedInventory` must not set a namespace; NKP generates one per cluster, and a hard-coded namespace breaks the SSH key secret.

## Air-gapped install

Download `nkp-air-gapped-bundle_v<version>_linux_amd64.tar.gz` from the Nutanix portal on a connected machine. For 2.17 it is on the order of 15 GB and unpacks to:

| Path | Role |
| --- | --- |
| `container-images/konvoy-image-bundle-*.tar` | Kubernetes, CAPI, CNI, CSI images |
| `container-images/kommander-image-bundle-*.tar` | Kommander, Prometheus, Grafana, Traefik, Dex, Flux |
| `container-images/nkp-catalog-applications-image-bundle-*.tar` | Optional catalog (Harbor, Istio, …) |
| `application-repositories/kommander-applications-*.tar.gz` | Helm charts Flux installs from |
| `konvoy-bootstrap-image-*.tar` | The kind image. `docker load` it. The bastion cannot pull it. |
| `nkp`, `kubectl` | CLIs |
| `kib/` | Image builder, if you build your own node image |

Production uses a registry (Harbor, Artifactory, Nexus). Push once, create many clusters.

```bash
docker load --input konvoy-bootstrap-image-v2.17.1.tar

nkp push bundle \
  --bundle ./container-images/konvoy-image-bundle-v2.17.1.tar \
  --bundle ./container-images/kommander-image-bundle-v2.17.1.tar \
  --to-registry=https://registry.site.example \
  --to-registry-username="${REGISTRY_USERNAME}" \
  --to-registry-password="${REGISTRY_PASSWORD}" \
  --to-registry-ca-cert-file=registry-ca.crt

nkp create cluster nutanix \
  <the same infrastructure flags as connected> \
  --self-managed \
  --airgapped \
  --registry-mirror-url=https://registry.site.example \
  --registry-mirror-username="${REGISTRY_USERNAME}" \
  --registry-mirror-password="${REGISTRY_PASSWORD}" \
  --registry-mirror-cacert=registry-ca.crt
```

Do not also pass `--bundle`. Bundle mode loads tarballs onto each node and is for a single lab cluster. Registry mode rewrites containerd on every node to the mirror.

The Nutanix VM image is a separate artifact. Import it into Prism before `create cluster`. The air-gap bundle does not contain it.

Harbor messages like `project not found, name: mesosphere` during `nkp push bundle` are noisy. The CLI creates the projects and continues.

Kommander charts, if you install or upgrade them by hand:

```bash
nkp install kommander --init --airgapped > kommander.yaml
nkp install kommander \
  --installer-config kommander.yaml \
  --kubeconfig "${CLUSTER_NAME}.conf" \
  --kommander-applications-repository \
    ./application-repositories/kommander-applications-v2.17.1.tar.gz
```

Run that from the extracted bundle root. The CLI uses relative paths.

Verify the mirror is what nodes actually use:

```bash
ssh "${SSH_USERNAME}@<node>" 'sudo grep -A5 mirror /etc/containerd/config.toml'
kubectl get events -A | grep -i pull
kubectl -n kommander wait --for=condition=Ready helmreleases --all --timeout=20m
```

## Upgrade

One NKP minor at a time. Management cluster before any managed cluster. Platform apps before the Kubernetes version they must support.

### 1. Bits

- New `nkp` binary, the one for the target version.
- New air-gap bundles pushed to the same registry, if the site is dark.
- New VM image in Prism, original filename. `nkp upgrade cluster nutanix` looks up the image by name.

### 2. Kommander, then workspaces

```bash
export KUBECONFIG=nkp-mgmt.conf

# connected
nkp upgrade kommander

# air-gapped, from the bundle root
nkp upgrade kommander \
  --kommander-applications-repository ./application-repositories/kommander-applications-vX.Y.Z.tar.gz

nkp get workspaces
nkp upgrade workspace "${WORKSPACE_NAME}"
```

The management cluster's own platform apps move with `nkp upgrade kommander` (the `kommander` workspace). `nkp upgrade workspace` moves managed and attached clusters in that workspace. Do every workspace before you change the Kubernetes version. Old platform apps are not compatible with the Kubernetes version in the new NKP release.

Between those two commands the management copies are ahead of the cluster copies. Flux, Dex, and Thanos all do this. The cluster's own Flux is the one pulling that cluster's `HelmRelease` and `OCIRepository` objects. The management Flux is the one that installed it. Compare the image tags on both contexts before you call a CVE done. The windows and the commands are in [Compatibility](../compatibility.md).

During this step the UI and the custom dashboard hostname can blip. Tokens for attached clusters are reissued. Users need a fresh kubeconfig from the Kommander endpoint after the upgrade.

On 2.19, for a non-ClusterClass cluster, `nkp upgrade platform-apps` is the independent platform-app step.

### 3a. Nutanix cluster upgrade

This is a CAPX rollout, not a kubeadm upgrade of the existing VM. `--vm-image` creates a new `NutanixMachineTemplate` aimed at that Prism image and sets `KubeadmControlPlane.spec.version` / the MachineDeployment version to the Kubernetes version baked into the image. Control-plane VMs are replaced first, then worker pools. Core add-ons (CNI, CSI) move in the same command.

You cannot point this command at the currently running image and expect only the kubelet to change. The image has to be the one built for the target Kubernetes version, already imported into Prism under its original name. A same-version image (newer OS packages, same Kubernetes version in the name) still replaces every VM. Patching the live VM with `dnf`, or editing it in Prism, is discarded on the next rollout. Pre-provisioned clusters are the exception: CAPPP does not clone VMs, so Kubernetes and the OS move separately and both happen on the host you already have.

The objects, the other NKP providers, and how this differs from OpenShift, GKE, EKS, AKS, and RKE2 are in [Cluster API](../capi.md).

```bash
export MANAGEMENT_CLUSTER_NAME="$(kubectl get -n kommander nkpcluster \
  -l 'kommander.d2iq.io/host=true' \
  -o jsonpath='{.items[0].metadata.name}')"

nkp upgrade cluster nutanix \
  --cluster-name "${MANAGEMENT_CLUSTER_NAME}" \
  --vm-image "${VM_IMAGE_NAME}"
```

Then each managed cluster:

```bash
kubectl get nkpclusters -A
nkp upgrade cluster nutanix \
  --cluster-name "${WORKLOAD_CLUSTER_NAME}" \
  --vm-image "${VM_IMAGE_NAME}" \
  -n "${WORKLOAD_CLUSTER_NAMESPACE}"
```

Different images for control plane and workers:

```bash
nkp get nodepools --cluster-name "${CLUSTER_NAME}" -n "${NS}"
nkp upgrade cluster nutanix \
  --cluster-name "${CLUSTER_NAME}" \
  -n "${NS}" \
  --control-plane-vm-image "${CP_IMAGE}" \
  --worker-vm-images "${POOL1}=${POOL1_IMAGE},${POOL2}=${POOL2_IMAGE}"
```

Worker pools upgrade one at a time. To run two at a time:

```bash
kubectl annotate cluster "${CLUSTER_NAME}" -n "${NS}" \
  topology.cluster.x-k8s.io/upgrade-concurrency=2
```

Respect PodDisruptionBudgets. A budget that forbids any eviction stalls the MachineDeployment, and the CLI looks hung.

### 3b. Other providers (aws, azure, vsphere, gcp, eks, preprovisioned)

The guide splits what the Nutanix command combines:

```bash
nkp upgrade capi-components
nkp upgrade addons aws --cluster-name "${CLUSTER_NAME}"
# then the Kubernetes version upgrade for that provider, e.g.
nkp upgrade cluster eks --cluster-name "${CLUSTER_NAME}"
```

`nkp upgrade addons` rewrites the `ClusterResourceSet` for CSI, CNI, cluster-autoscaler, and node-feature-discovery. Reapply any hand edits from a dry-run:

```bash
nkp upgrade addons aws --cluster-name "${CLUSTER_NAME}" --dry-run -o yaml > addons.yaml
# patch addons.yaml, then
kubectl apply -f addons.yaml
```

### What to watch

```bash
kubectl get clusters,machines,machinedeployments -A
kubectl get helmreleases -n kommander
kubectl get clusterresourcesets -A
kubectl get pods -n ntnx-system
kubectl get pods -A -l app.kubernetes.io/name=cilium
kubectl get pods -n calico-system
```

A new `Machine` in `Provisioning` is the upgrade. An old `Machine` stuck in `Deleting` is a drain that cannot finish (PDB, a volume that will not detach, a finalizer).

## CNI day-2

Bundled CNI config is the Helm values inside the `ClusterResourceSet` ConfigMap. Changing them rolls the CNI DaemonSet (speed 2) and is wiped on the next `nkp upgrade addons` unless you reapply.

Replacing Cilium with Calico, or the reverse, after the cluster is up is not an upgrade. Drain the cluster onto a new one, or follow the create-time removal above on a fresh cluster.

Useful checks:

```bash
kubectl -n kube-system rollout status ds -l app.kubernetes.io/name=cilium
cilium status --wait
```

`cilium` CLI is only present if you installed it. The DaemonSet ready count is the check that always works.

## CSI day-2

On AHV clusters NKP installed, CSI is a core add-on. You get the version in that NKP release, applied by `nkp upgrade addons` or by `nkp upgrade cluster nutanix`. A StorageClass change is immediate and does not upgrade the driver:

```bash
kubectl get storageclass
kubectl get csidriver
kubectl get pods -n ntnx-system
```

On pre-provisioned clusters where you installed the Helm chart, you own the version:

```bash
helm upgrade nutanix-csi ./nutanix-csi-storage -n ntnx-system \
  --reuse-values \
  --version <chart version matched to this NKP minor>
```

Test a PVC, a pod mount, an expansion, and a delete. Nutanix Volumes need the iSCSI daemon on the node. A CSI pod that is Ready and a volume that never attaches is usually iSCSI or a Prism permission, not the Helm release.

Nutanix Data Services for Kubernetes (NDK) is a separate catalog bundle (`nutanix-product-catalog-*-airgapped.tar`). It is only for Nutanix infrastructure. Do not push it into an AWS or pre-provisioned registry and expect those clusters to use it.

## At scale

| Knob | Effect |
| --- | --- |
| Workspace | `nkp upgrade workspace` upgrades platform apps on every cluster in it. Split nonprod and prod into different workspaces. |
| One `nkp upgrade cluster` | One cluster's machines and its core add-ons. |
| `upgrade-concurrency` | How many worker pools inside that cluster move at once. Default 1. |
| ClusterClass, NKP 2.19+ | One UI action or one CLI command per workload cluster, platform apps before Kubernetes. Management cluster is already on 2.19. |
| GitOps-managed workload clusters | Flux owns the cluster object. The upgrade guide has a different sequence. Do not also run `nkp upgrade cluster` against a cluster Flux will revert. |

A reasonable prod loop is: Kommander once, nonprod workspace, one nonprod cluster, soak, then a serial loop over prod `NKPCluster` objects. More in [Upgrade at scale](../scale.md).

## Traps

- Combining `--bundle` and `--registry-mirror-*`. Pick one.
- Renaming the Prism image. The upgrade looks up the name the release used.
- Editing a `ClusterResourceSet` in the UI and expecting it to survive `nkp upgrade addons`.
- Upgrading the Kubernetes version before `nkp upgrade workspace`. Platform apps break on the new APIs.
- Forgetting `docker load` of the bootstrap image in an air gap. The failure looks like Docker Hub being unreachable, because it is.
- A bundle-mode Kommander install that never creates `host-cluster-oci-creds`, so every `OCIRepository` fails TLS. Registry-mirror installs avoid that class of bug. If you are on bundle mode and Flux cannot pull, check that secret and `oci-install-config` in the `kommander` namespace.
- Pre-provisioned CSI left on the local-volume default StorageClass. Apps look "installed" and have no real volumes.
- Editing `NutanixMachineTemplate` in place, or the VM in Prism, and expecting a roll. Templates are immutable. Prism edits are drift. Create a new template (or pass a new `--vm-image`) and let CAPX replace the VM.
- Bumping `KubeadmControlPlane.spec.version` while leaving the old image pinned. Raw Cluster API will clone that image and try to install the new Kubernetes on it. NKP images are not built for that. Import the matching image.
- Treating pre-provisioned like CAPX. There is no replacement VM. A host that cannot boot the new kubelet stays broken until you fix the OS yourself.
