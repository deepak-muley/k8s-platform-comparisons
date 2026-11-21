# RKE2 and Rancher

RKE2 is a Kubernetes distribution packaged as a single systemd service and an air-gap image tarball. Rancher Manager is the fleet UI in front of many RKE2 (or other) clusters. Fleet, in the Rancher sense, is GitOps for manifests across those clusters. It is not the same word as an AKS fleet or a GKE fleet.

This page is here because RKE2 is the usual on-prem comparison point for NKP and EKS Anywhere: you own the nodes, the air gap is a tarball plus a registry, and CSI is always yours.

RKE2 itself is not Cluster API. The new Kubernetes version is a package on the host you already have, and the OS package manager is a different package. You can move either one and leave the other. That is the operation CAPX will not do. It comes back only if Rancher provisioned the nodes: those machine pools replace VMs from an infrastructure template, and an image baked for one Kubernetes version couples them again. Registered nodes stay on the in-place RKE2 model. See [Cluster API](../capi.md).

## Core vs add-on

| | What | Where it lives | Moves when | Speed |
| --- | --- | --- | --- | --- |
| Core | API server, etcd, scheduler, controller-manager, kubelet | `rke2-server` on control-plane nodes, `rke2-agent` on workers. Config in `/etc/rancher/rke2/config.yaml`. | Install the new RKE2 version and restart the service. Servers one at a time, then agents. | 3 |
| Platform core, if you use the default | Canal (Calico + Flannel), CoreDNS, and the other charts RKE2 enables by default | HelmCharts applied from `/var/lib/rancher/rke2/server/manifests` | With the RKE2 version. The chart manifest is inside the release. | 3 for the node restart, 2 for the chart pods that roll during it |
| Platform core, your choice at install | Cilium or Calico, if `cni:` is set | The same manifests directory, generated for that CNI | With the RKE2 version, as long as RKE2 owns the chart | 3 |
| Add-on | A CNI you installed because `cni: none`, and every CSI | Your Helm release, or a `HelmChart` manifest you put in the manifests directory | `helm upgrade`, any day | 2 |
| Fleet layer | Rancher itself, and Fleet `GitRepo` bundles | The Rancher management cluster | Upgrade Rancher before you upgrade downstream Kubernetes minors | 2 on the management cluster |

`cni:` is a create-time choice if RKE2 owns the CNI. Changing `cni: canal` to `cni: cilium` on a live cluster is a rebuild. The supported way to own the CNI cadence is `cni: none` at install, then Helm-install Cilium yourself and never let RKE2's bundled chart take the CNI socket.

RKE2 does not ship a cloud CSI. There is no default StorageClass that provisions real volumes until you add one. The local-path provisioner some examples install is a lab default. Treat it the way NKP's local-volume class should be treated: fine for a test, wrong for a disk you meant to survive a node.

## Install one node, then the rest

Connected, first server:

```bash
curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_VERSION=v1.32.3+rke2r1 sh -
sudo systemctl enable --now rke2-server
sudo cat /etc/rancher/rke2/rke2.yaml          # admin kubeconfig, server is 127.0.0.1
sudo cat /var/lib/rancher/rke2/server/node-token
```

`/etc/rancher/rke2/config.yaml` before you start the service, if you already know the CNI and the registry:

```yaml
write-kubeconfig-mode: "0640"
cni: cilium
system-default-registry: registry.site.example:5000
tls-san:
  - api.rke2.site.example
cluster-cidr: 10.42.0.0/16
service-cidr: 10.43.0.0/16
```

Join another server (HA etcd wants three):

```yaml
server: https://api.rke2.site.example:9345
token: <node-token>
cni: cilium
system-default-registry: registry.site.example:5000
```

```bash
sudo systemctl enable --now rke2-server
```

Agents use the same config with `rke2-agent` instead of `rke2-server`. They do not run etcd.

Put `kubectl` on your PATH from the RKE2 bundle (`/var/lib/rancher/rke2/bin`) and use `/etc/rancher/rke2/rke2.yaml` as the kubeconfig. The API is on 6443. Node join is on 9345.

Pod CIDR and service CIDR are create-time. `cluster-cidr` after etcd has been running is a new cluster.

## Air gap

On a connected machine, pull the exact release. The `+` in the version is `%2B` in the URL.

```bash
VER=v1.32.3%2Brke2r1
curl -LO "https://github.com/rancher/rke2/releases/download/${VER}/rke2.linux-amd64.tar.gz"
curl -LO "https://github.com/rancher/rke2/releases/download/${VER}/rke2-images.linux-amd64.tar.zst"
curl -LO "https://github.com/rancher/rke2/releases/download/${VER}/sha256sum-amd64.txt"
curl -LO "https://github.com/rancher/rke2/releases/download/${VER}/install.sh"
```

Check the checksum, carry the files in, and on the first server:

```bash
sudo mkdir -p /var/lib/rancher/rke2/agent/images /root/rke2
sudo cp rke2-images.linux-amd64.tar.zst /var/lib/rancher/rke2/agent/images/
sudo cp rke2.linux-amd64.tar.gz install.sh /root/rke2/
sudo INSTALL_RKE2_ARTIFACT_PATH=/root/rke2 sh /root/rke2/install.sh
sudo systemctl enable --now rke2-server
```

For every further node, prefer a registry. Load the images tarball into `registry.site.example` once, set `system-default-registry`, and stop copying a multi-gigabyte file by hand. The tarball path remains the fallback when a node cannot pull yet.

RKE2's images tarball contains the Kubernetes and bundled-CNI images. It does not contain the CSI chart you have not chosen, and it does not contain Rancher. A Rancher air gap is a second, larger mirror (the Rancher images list for that Rancher version). Do it before you promise a management UI inside the site.

Private CA: write `/etc/rancher/rke2/registries.yaml` before the service starts. This is the file RKE2's containerd actually reads.

```yaml
mirrors:
  registry.site.example:5000:
    endpoint:
      - "https://registry.site.example:5000"
configs:
  registry.site.example:5000:
    tls:
      ca_file: /etc/rancher/rke2/registry-ca.pem
```

Copy the CA to that `ca_file` path on every node. A node that is `NotReady` with `ImagePullBackOff` on `cilium` or `canal` is almost always this file missing, or `system-default-registry` missing on that one node. Restart `rke2-server` or `rke2-agent` after you add it.

## Day-2 config

RKE2 reconciles the manifests in `/var/lib/rancher/rke2/server/manifests` on the servers that have the manifests. Edit a HelmChart there when you want RKE2 to keep applying it. Edit `/etc/rancher/rke2/config.yaml` and restart the service when the change is a kubelet or apiserver flag.

Restarting `rke2-server` restarts the control plane on that node. One server at a time, and wait until etcd and the API are healthy:

```bash
sudo systemctl restart rke2-server
kubectl get nodes
kubectl get pods -n kube-system
```

A `HelmChart` you added for a CSI driver, for example, lives in that directory and rolls only the chart's pods. That is the speed-2 path. Do not put day-2 tweaks into a bundled manifest RKE2 will overwrite on the next version bump. Put them in a file you own, with a name that does not collide with the packaged charts.

## Upgrade RKE2

One minor at a time. All servers to the new version before any agent. etcd members that span too many RKE2 versions are how HA clusters lose quorum during an upgrade.

Air-gapped, stage the new tarball and the new images first. Then, on server 1:

```bash
sudo INSTALL_RKE2_VERSION=v1.32.4+rke2r1 \
  INSTALL_RKE2_ARTIFACT_PATH=/root/rke2 \
  sh /root/rke2/install.sh
sudo systemctl restart rke2-server
kubectl get nodes
# wait until this node is Ready and etcd has a leader
```

Then server 2, then server 3, then agents in batches a PDB can survive. Agents:

```bash
sudo systemctl restart rke2-agent
```

Rancher-managed downstream clusters do the drain and the restart for you if you change the Kubernetes version on the cluster object. The concurrency setting on that cluster is the surge. Still keep it at one for the first prod cluster of a new minor.

Bundled charts move with the RKE2 version. Your CSI Helm release does not. Plan a chart bump as its own change, either before the minor (so the chart already understands the new APIs) or after, not silently inside it.

## CNI

RKE2-owned Cilium, set at install:

```yaml
cni: cilium
```

The version is the one in that RKE2 release. You get a newer Cilium by upgrading RKE2, which restarts nodes.

Self-owned Cilium, the fast path:

```yaml
cni: none
```

Install Cilium with Helm after the first server is up and before you expect normal pods to schedule. System pods that tolerate a missing CNI can be configured; ordinary workloads cannot. From then on:

```bash
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --version "${CILIUM_VERSION}" \
  --reuse-values
```

That is speed 2. It does not restart `rke2-server`. Keep the chart's Kubernetes compatibility in mind when you later bump RKE2. A Cilium release that is too old for the new kubelet is the failure mode of owning it yourself, and it is a better failure mode than being unable to patch Cilium for four months.

Do not run Canal and Cilium. `cni: none` means RKE2 must not also deploy Canal. Check the manifests directory if pods have two CNI binaries fighting:

```bash
sudo ls /var/lib/rancher/rke2/server/manifests
kubectl get ds -A
```

## CSI

Nothing is installed. Pick the driver that matches the disks you have.

```bash
helm upgrade --install nutanix-csi ./nutanix-csi-storage \
  --namespace ntnx-system --create-namespace \
  -f csi-values.yaml
```

or the vSphere CSI chart, or Longhorn, or Ceph. Longhorn and Ceph are popular on RKE2 because the cluster is often on hardware with no external array. They are still add-ons. They have their own upgrade, and a Longhorn upgrade while an RKE2 node restart is in progress is two storage disruptions at once. Finish one before you start the other.

StorageClass, snapshot class, and a default-class annotation are speed 1. Remove any local-path default annotation when you add a real class.

Test across a node restart, not only across a Helm roll. RKE2 upgrades restart the node. A volume plugin that cannot reattach after `rke2-agent` comes back is the bug you want to find on the canary.

## At scale with Rancher

Rancher runs on its own management cluster (often a small RKE2 cluster, sometimes upstream Kubernetes). Downstream clusters are `clusters.management.cattle.io` objects, or the cluster resource in the UI.

- Upgrade Rancher first, to a version that supports the Kubernetes minor you are about to install on downstream clusters. The support matrix is per Rancher minor. Running a downstream minor newer than the management Rancher understands fails in provisioning, not in a clean preflight.
- Canary: one downstream cluster, concurrency 1, RKE2 servers before agents.
- Prod: raise concurrency only for node drains that you have watched succeed. A concurrency of 10 on a 12-node cluster is a full outage if the new RKE2 version is bad.
- Apps and "platform" charts (monitoring, ingress, compliance) belong to Fleet `GitRepo` bundles with a target selector. The canary target matches `env: nonprod`. The prod target matches nothing until you change it. Fleet rolling out a bad chart is speed 2 and is much easier to revert than an RKE2 minor. Keep those bundles out of the same Git commit as a Kubernetes upgrade.

Air-gapped Rancher uses a private registry for the Rancher image list and the system charts. Downstream RKE2 clusters still need their own `system-default-registry` or their own image tarball. Mirroring Rancher does not mirror the RKE2 version a downstream cluster is about to upgrade to. Stage both.

## Traps

- Restarting all three `rke2-server` nodes together. etcd loses quorum. The cluster comes back only if you are lucky with timing.
- `cni` different in one node's `config.yaml` than in the others. That node joins and never becomes Ready, or it becomes Ready and blackholes traffic.
- Editing a packaged manifest under `/var/lib/rancher/rke2/server/manifests` and being surprised when the next RKE2 version replaces the file.
- A local-path default StorageClass left in place after the real CSI was installed. Half the PVCs are hostPath.
- Upgrading RKE2 and Longhorn in one window.
- An air-gap node missing `system-default-registry` that silently falls through to Docker Hub and fails later, when the site firewall is finally closed.
- Rancher a minor behind the downstream clusters it is trying to provision. Check the matrix before the Fleet run, not during it.
