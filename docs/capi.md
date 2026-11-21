# Cluster API, and whether Kubernetes and the OS can move apart

NKP on Nutanix does not upgrade a node. CAPX (Cluster API Provider Nutanix) deletes the VM and creates another one from a `NutanixMachineTemplate`. The template points at a Prism image, and an NKP image is built for one Kubernetes version. That is the restriction: the supported upgrade moves the OS image and the kubelet together, and it moves them by replacement.

The other products on this site do not all share that restriction. Some replace VMs but can move the OS image alone. Some reboot the same machine. One NKP provider, pre-provisioned, cannot create a VM at all.

## The two knobs

Cluster API keeps them in different objects.

| Knob | Object and field | What it selects |
| --- | --- | --- |
| Kubernetes version | `KubeadmControlPlane.spec.version`, and `MachineDeployment.spec.template.spec.version` | The version the bootstrap provider (kubeadm) installs on a machine |
| OS image | The infrastructure template. On Nutanix that is `NutanixMachineTemplate.spec.template.spec.image` or `imageLookup`. | The disk the hypervisor clones |

Stock Cluster API applies either change by creating a Machine and deleting the old one. It does not SSH in and run `kubeadm upgrade` or `dnf update` on the VM you already have. Editing the live VM in Prism, or patching packages on it, is drift. The next scale or rollout builds a new VM from the template and the drift is gone.

Templates are immutable. Changing `image.name` on an existing `NutanixMachineTemplate` does not roll anything. The working sequence is: create a new template with a new name, then point the control plane or the MachineDeployment at that name.

```bash
# what a rollout is actually doing
kubectl get kubeadmcontrolplanes,machinedeployments,machines,nutanixmachinetemplates -A
```

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: NutanixMachineTemplate
metadata:
  name: prod-mt-1-35-2          # new name. do not edit prod-mt-1-34-1 in place
  namespace: demo-zone
spec:
  template:
    spec:
      image:
        type: name
        name: nkp-rocky-9.7-release-cis-1.35.2-20260626061237.qcow2
      cluster:
        name: prism-element-01
      subnet:
        - name: workload
---
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlane
metadata:
  name: prod-control-plane
  namespace: demo-zone
spec:
  version: v1.35.2
  replicas: 3
  infrastructureTemplate:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: NutanixMachineTemplate
    name: prod-mt-1-35-2
  rolloutStrategy:
    rollingUpdate:
      maxSurge: 1
```

Workers are the same idea on the MachineDeployment: `spec.template.spec.version` plus `spec.template.spec.infrastructureRef.name`. `nkp upgrade cluster nutanix --vm-image` writes both. On a ClusterClass cluster (NKP 2.19 topology), edit the `NKPCluster` or the class variables. The topology controller regenerates the MachineDeployment. A hand edit of the generated object is reverted.

CAPX also accepts `imageLookup` instead of a pinned image, not both:

```yaml
imageLookup:
  baseOS: ubuntu-22.04
  format: capx-{{.BaseOS}}-{{.K8sVersion}}-*
```

The default format is `capx-{{.BaseOS}}-{{.K8sVersion}}-*`. The version comes from the Machine. A bump of `spec.version` to `v1.35.2` makes CAPX search Prism for `capx-ubuntu-22.04-1.35.2-*`. If that image is not there, the new Machine stays `Provisioning`. Lookup does not fall back to the previous image.

## What "upgrade only Kubernetes" can mean

Three different operations get called that. Only the first is what people usually want, and CAPX-on-NKP does not do it.

| Operation | Same VM | OS image | Who actually does this |
| --- | --- | --- | --- |
| New kubelet on the disk that is already booted | Yes | Unchanged | RKE2 (`systemctl restart` after installing the new version). NKP pre-provisioned, because CAPPP cannot build a replacement VM. OpenShift does the opposite coupling: it reboots the same VM, but the new ostree commit contains both RHCOS and the kubelet. |
| New VM cloned from the same image, kubeadm installs a different Kubernetes version | No | The template's image is unchanged | Raw Cluster API, when the image is a generic OS whose bootstrap downloads Kubernetes. Not the NKP supported path. An NKP image is named for one Kubernetes version, and the upgrade guide requires the image to match that version. |
| New VM cloned from a new image that already contains the new kubelet | No | New image, built for that Kubernetes version | NKP + CAPX. EKS Anywhere. EKS-optimized AMI. GKE node version. AKS `az aks upgrade`. |

So on an NKP Nutanix cluster you do not get a Kubernetes upgrade that leaves the OS image where it is. The image that matches the new Kubernetes version has to be in Prism first, and every control-plane VM and then every worker VM is replaced.

The reverse, a new OS image with the same Kubernetes version, is a CAPI operation (new template, same `spec.version`) and still replaces every VM. NKP will do it only when you have an image built for that same Kubernetes version. Portal images are cut per NKP release and per Kubernetes version (`nkp-rocky-9.7-release-cis-1.35.2-…`). A newer OS build at the same Kubernetes version exists when Nutanix published one. There is no `--node-image-only` switch. `dnf update` on the running VM is not that operation.

## What replacement costs

Budget these on every CAPX rollout, including an "OS only" one.

- **Spare capacity.** `maxSurge: 1` needs one extra VM, one address, and disk in the Prism storage container before the old VM is deleted. `maxUnavailable: 0` with no room to surge sits there. Control planes are rolled one at a time so etcd keeps quorum. A 3-node control plane can lose one member. It cannot lose two.
- **A new Machine identity.** New `providerID`, new node name, new NIC. Pods are rescheduled, not restarted in place. hostPath, emptyDir, and local PVs die with the old disk. A Nutanix volume can move only if the CSI node plugin attaches it to the new VM. Test that on the canary. An iSCSI session does not follow a VM that no longer exists.
- **etcd membership changes** on every control-plane replace. An OpenShift master reboot keeps the etcd data directory. A CAPX replace removes a member and adds one. Take the backup before the roll, not after the first new VM is up.
- **PDBs.** The new VM is Ready, the old one stays `Deleting`, and the MachineDeployment looks hung. The surge VM is still consuming the spare capacity, so the next pool does not start either.
- **Prism is not the source of truth.** CPU, memory, a disk you added in Prism, a package you installed: gone on the next rollout. Change the template.
- **The image is not in the container air-gap bundle.** Import the qcow into Prism under the name the release used. `nkp upgrade cluster nutanix` looks the image up by that name. Renaming it fails the rollout in a way that looks like a missing template.
- **Scale-up uses the current template.** After the upgrade, a new worker is cloned from the new image. If lookup cannot see that image, or the pinned image was deleted, scale fails even though the cluster is Ready.
- **Preflight is per NKP version.** cgroup v2 is required. A custom image has to be built with the same `nkp` binary you will upgrade with.

Cluster API v1.12 (January 2026) can apply some Machine changes in place when the `InPlaceUpdates` and `RuntimeSDK` feature gates are on and an extension claims the change. If the extension does not claim it, Cluster API replaces the Machine, which is still the default. An OS image swap is the change the project itself treats as a rollout. NKP 2.17–2.19 upgrades do not go through that extension. `nkp upgrade cluster nutanix` provisions new VMs. Upstream chained upgrades (declare 1.35 and let Cluster API walk 1.33 and 1.34) are also not an NKP feature. NKP is one minor at a time because the CLI, the Kommander apps, and the image are one release.

## NKP providers

`nkp` is one CLI over several infrastructure providers. The Kubernetes-versus-OS rule follows the provider, not the CLI.

| `nkp` provider | Infrastructure API | What an upgrade does to a node | Kubernetes without a new OS image | OS image without a new Kubernetes version |
| --- | --- | --- | --- | --- |
| `nutanix` | CAPX. `NutanixMachineTemplate` image or `imageLookup`. | New AHV VM, then delete the old one. | No. The image must match the Kubernetes version of this NKP release. Lookup selects the image from the version, so the old image is not a legal result. | Only by rolling onto another image built for that same Kubernetes version. Still a new VM. |
| `preprovisioned` | CAPPP. Inventory of hosts you already installed. | Stops at the host. Nothing is cloned. | Yes. Bootstrap moves Kubernetes on that host. The host has to already meet the new kubelet (kernel, cgroup v2, uploaded artifacts). | Yes, and it is your job. Patch the OS with the OS tools. NKP will not. A dead host stays dead until you replace it in the inventory. |
| `aws`, `azure`, `vsphere`, `gcp` | CAPA, CAPZ, CAPV, CAPG. | New instance or VM from the provider template (AMI, Azure image, vSphere template, GCP image). | Same product rule as Nutanix. Konvoy Image Builder images are cut per Kubernetes version. Raw Cluster API could reuse a generic image and let kubeadm install Kubernetes. The NKP-supported image is not generic. | New template, same `spec.version`, if an image for that exact Kubernetes version exists. |
| `eks` | EKS for the control plane. Node groups for the workers. | Control plane: EKS updates it. No VM to replace. Workers: node group replacement, EKS AMI rules. | Control plane version moves without an AMI you manage. Workers move to the AMI EKS associates with the target version. | A new worker AMI at the same Kubernetes version is an EKS node group property, not a CAPX image. See [EKS](platforms/eks.md). |

`nkp upgrade capi-components` on the management cluster is the step that upgrades the controllers (CAPI, CAPX, CAPA, …). Do it before the workload-cluster roll on the non-Nutanix providers. The Nutanix management-cluster command includes it. Old controllers will happily try to honor a new template they do not understand.

## The other platforms

| Platform | Engine | Same machine or a new one | Kubernetes moves, OS image stays | OS image moves, Kubernetes version stays |
| --- | --- | --- | --- | --- |
| NKP on CAPX | Cluster API | New VM | No | Only with a same-version image. New VM either way. |
| NKP pre-provisioned | Cluster API, host boundary | Same host | Yes | Yes. You patch it. |
| EKS Anywhere | Cluster API. CAPV, CAPX, Tinkerbell, or Snow. | New machine. Bare metal reprovisions the host and wipes the disk. Surge needs a spare host. | The published image matches one `kubernetesVersion`. Changing the version without that image does not keep the old OS install. | A new template or `osImageURL` at the same `kubernetesVersion`, if that image exists. Still a reprovision. |
| OpenShift | Machine API creates VMs. The Machine Config Operator updates the OS. | Same VM, rpm-ostree, reboot. The VM is replaced when it is scaled or when the machine spec itself changes (size, subnet), not because the payload moved. | No. A z-stream ships a new RHCOS commit and the matching kubelet together. | No separate OS channel. The z-stream is the OS update, and the kubelet patch comes with it. Local disks survive the reboot. etcd membership does not change. |
| GKE | Google's node-pool controller, not Cluster API | New GCE VM (surge or blue-green) | No. The node version is one string, `1.32.8-gke.1056000`. The image and the kubelet are that artifact. | Yes, inside the minor. A newer `-gke.N` build is a new image at the same Kubernetes patch line. The VM is still new. You cannot `apt` the existing node. |
| EKS | AWS updates the control plane. Node groups and Karpenter replace instances. | Control plane: no instance you own. Nodes: new EC2 instance. | Control plane: yes, `update-cluster-version` does not touch AMIs. Nodes: no. The EKS-optimized AMI is built for one Kubernetes version. | Yes. A new AMI for that same Kubernetes version, then `update-nodegroup-version` without a cluster-version change. New instances. |
| AKS | AKS resource provider, not Cluster API | New VM via surge, old VM deleted | No. `-k` moves the Kubernetes version and the node image together. | Yes. `az aks nodepool upgrade --node-image-only`, or a Fleet run with `--upgrade-type NodeImageOnly`. New VMs, same Kubernetes version. |
| RKE2 | systemd on a host you installed | Same host | Yes. Install the new RKE2 version, restart the unit. | Yes. Patch the OS. Reboot if the kernel changed. The RKE2 version stays. |
| Rancher, machines Rancher provisioned | Cluster API under the machine pool | New VM when the pool's infrastructure template changes | Follows the image. A generic cloud image plus RKE2-at-boot can separate them. An image baked for one Kubernetes version cannot. | New template at the same Kubernetes version, if you have that image. |
| Rancher, registered nodes | The node is a host you added | Same host | Yes, same as RKE2. | Yes, same as RKE2. |

## How to choose the operation

On CAPX, decide which image is going into Prism before you pick a Kubernetes version. The version you can land on is the set of images you have actually imported.

```bash
# Prism image name, exactly as imported
export VM_IMAGE_NAME=nkp-rocky-9.7-release-cis-1.35.2-20260626061237.qcow2

nkp upgrade cluster nutanix \
  --cluster-name "${CLUSTER_NAME}" \
  -n "${CLUSTER_NAMESPACE}" \
  --vm-image "${VM_IMAGE_NAME}"
```

Watch the replacement, not a kubelet restart:

```bash
kubectl get machines -n "${CLUSTER_NAMESPACE}" -o wide
kubectl get kubeadmcontrolplane,machinedeployment -n "${CLUSTER_NAMESPACE}"
```

A healthy roll is a new Machine in `Provisioning`, then `Running`, then the old Machine in `Deleting`. A Machine that stays `Provisioning` is usually the image (name mismatch, lookup miss, Prism capacity) or the bootstrap (the image's kubelet does not match `spec.version`). A Machine that stays `Deleting` is a drain: PDB, a volume that will not detach, a finalizer.

On GKE, AKS, or EKS, the OS-only week is a real, separate command (a `-gke.N` node-pool upgrade, `--node-image-only`, a new AMI at the same Kubernetes version). Do not look for that command on CAPX. Look for a second image whose name contains the same Kubernetes version.

On RKE2 and on NKP pre-provisioned, the OS-only week is the OS package manager, and the Kubernetes-only week is the distro package. They are already separate. Putting those hosts under a CAPI infrastructure provider on purpose (CAPX, CAPV, CAPA) removes that separation, because the image becomes the unit again.
