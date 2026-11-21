# Connected vs air-gapped

An air-gapped cluster does not phone home for images, charts, or the upgrade graph. The bastion, or a one-way transfer into the site, is the only place that ever had internet. After that, every pull comes from a registry or a tarball you already staged.

Public GKE, public EKS, and public AKS are connected products. Their control planes live in the vendor cloud. The air-gap counterparts are different products: Google Distributed Cloud air-gapped, EKS Anywhere, and AKS on Azure Local. Do not plan an air-gap runbook for `gcloud` or `aws eks` and expect it to work with the registry mirror flags those CLIs do not have.

## The shape every real air-gap shares

1. On a connected machine, download a versioned bundle for the exact release you will install or upgrade to.
2. Scan it, transfer it, and load it into an internal registry. A tarball-on-every-node path exists for labs. It ages badly as soon as you have a second cluster.
3. Teach the nodes to trust the registry CA and to rewrite upstream registries to the mirror.
4. Install or upgrade by pointing at the mirror. The cluster must not be configured to ask the public update service.
5. For the next upgrade, repeat 1–3 before you change the desired version. If the bits are not in the mirror, the upgrade starts and then wedges on `ImagePullBackOff`.

Clock sync matters more in an air gap than on the internet. TLS to the registry and etcd certificates fail in confusing ways when the site NTP is wrong. Point every node at an internal NTP source before the first bootstrap.

## What has to be mirrored

| Platform | Images | Charts / manifests | Upgrade graph | Node OS |
| --- | --- | --- | --- | --- |
| OpenShift | Release payload, operator index, every operator you subscribe to | Operator catalogs are container images. Helm charts you added are extra. | Cincinnati graph, served by OpenShift Update Service inside the gap, or skip the graph and pass `--to-image` | RHCOS bootimage is inside the release payload. The Machine Config Operator pulls it through the mirror. |
| NKP | Konvoy bundle (Kubernetes, CAPI, CNI, CSI) and Kommander bundle (platform apps) | `application-repositories/*.tar.gz` for Kommander. Catalog and Insights bundles if you use them. | There is no phone-home graph. The new CLI plus the new bundle is the graph. | VM image imported into Prism Central, or the OS you already installed for pre-provisioned. The image is not inside the container bundle. |
| EKS Anywhere | `eksctl anywhere download images` | Curated packages are a separate import. | The new CLI knows the legal next versions. `upgrade plan` shows them. No public graph call is required after the CLI and images are local. | Bare metal: `osImageURL` and hook OS image, hosted inside the gap. vSphere/Nutanix: templates that already exist on the infrastructure. |
| RKE2 | The RKE2 air-gap image tarball for that exact version | Manifests you write. System charts are inside the images tarball. | You choose the next version by installing that binary. | The host OS packages, from an internal yum/apt mirror. RKE2 will not patch the OS for you. |
| GDC air-gapped | Staged by the GDC platform, not by `gcloud`. | Platform bundles on the appliance. | `UserClusterMetadata` lists the Kubernetes versions the management API already has. | Node images on the appliance. |
| EKS, GKE, AKS public | Not an air-gap design. Private clusters still pull add-on images from vendor registries, or from a pull-through cache you operate. A private cluster is a network control. It is not an air gap. | — | The vendor API is the graph, and it is on the internet. | Vendor-published, pulled by the managed node group. |

## Registry pattern

Use one highly available registry for every cluster in the site. Harbor, Artifactory, Nexus, or an internal ECR-style registry all work. The registry is in the data path for every scale-up and every upgrade. A laptop running `docker run registry` is how labs get stuck at 2 a.m.

Trust:

- Nodes need the registry CA in the container runtime trust store.
- OpenShift uses `image.config.openshift.io/cluster` `additionalTrustedCA`, and a cluster-wide proxy CA if the mirror is reached through a proxy.
- NKP takes `--registry-mirror-cacert` at create. `--additional-trust-bundle` has a 16 KB cloud-init limit. Pass the root and the intermediate, not a fat bundle.
- EKS Anywhere takes the mirror in `registryMirrorConfiguration`, including the CA and auth.
- RKE2 uses `system-default-registry` plus containerd config for the CA.

Auth:

- A pull robot that every node uses. Rotate it.
- OpenShift's pull secret (`~/.docker/config.json` merged into the global pull secret) must contain the mirror host.
- NKP registry username and password are cluster-create flags and end up in node config. Treat the bastion and the resulting secrets as production credentials.
- Do not embed the password in a git repo of cluster YAML. EKS Anywhere cluster specs and NKP flags both end up copied. Use a local untracked overlay or a secret store.

## Connected install, what "connected" still requires

Connected does not mean the cluster can be built from a laptop with no other infrastructure.

| Platform | What must already exist |
| --- | --- |
| OpenShift IPI | DNS records for `api.` and `*.apps.`, the platform account (AWS, vSphere, Nutanix, Azure, GCP), a pull secret from Red Hat, NTP. |
| OpenShift Assisted / Agent-based | The same DNS and pull secret. Agent-based is the one you want when the nodes cannot reach the internet but the admin can, or when you are shipping an ISO. |
| NKP | A bastion with Docker, the `nkp` CLI, and credentials for the infrastructure. For Nutanix: Prism Central, a subnet, a storage container, an NKP VM image already uploaded, a free IP for the control-plane endpoint, and a range for the Kubernetes service load balancer. |
| GKE | A project, a VPC, IAM. Autopilot needs less from you than Standard. |
| EKS | A VPC with subnets, an IAM role for the cluster, and either a managed node group role or Auto Mode. Add-ons need their own IAM roles (IRSA or pod identity) before the driver can talk to AWS APIs. |
| EKS Anywhere | An admin machine with Docker, the infrastructure (vSphere, Nutanix, Snow, or bare metal with Tinkerbell), and images the provider can clone. |
| AKS | A resource group, a VNet if you are not letting AKS make one, an identity for the cluster. |
| RKE2 | Three (or one, for a lab) Linux hosts, with the ports RKE2 needs open between them. |

## Air-gap install, the shortest path that still works in production

### OpenShift

On the connected mirror host:

```bash
oc-mirror --config imageset-config.yaml docker://registry.site.example:5000 --v2
```

`imageset-config.yaml` names the channel (`stable-4.19`), the min and max version, and the operator catalog heads you actually install (CSI, logging, GitOps). Mirror the operators in the same run as the payload. A payload-only mirror installs a cluster whose OperatorHub cannot install the CSI operator you need on day two.

Carry the `oc-mirror` workspace to the site. The results directory contains an `ImageDigestMirrorSet` (or `ImageContentSourcePolicy` on older releases) and a `release-signatures` directory.

During install, `install-config.yaml` points `imageContentSources` at the mirror and `additionalTrustBundle` at the CA. The Agent-based installer ISO is the usual vehicle when the nodes themselves are dark.

### NKP

```bash
tar -xzvf nkp-air-gapped-bundle_v2.17.1_linux_amd64.tar.gz
docker load --input konvoy-bootstrap-image-v2.17.1.tar

nkp push bundle \
  --bundle ./container-images/konvoy-image-bundle-v2.17.1.tar \
  --bundle ./container-images/kommander-image-bundle-v2.17.1.tar \
  --to-registry=https://registry.site.example \
  --to-registry-username="${REGISTRY_USERNAME}" \
  --to-registry-password="${REGISTRY_PASSWORD}" \
  --to-registry-ca-cert-file=registry-ca.crt
```

Then `nkp create cluster nutanix ... --airgapped --registry-mirror-url=...`. The Nutanix VM image is a separate download and must already be in Prism Central. The container bundle does not contain it.

For a single lab cluster, `--bundle ./container-images/konvoy-image-bundle-....tar,...` replaces the registry flags. Do not combine `--bundle` with `--registry-mirror-*`. The bundle path loads images onto each node and is the wrong tool once you have more than one cluster or you care about upgrades.

Kommander in an air gap needs the chart tarball:

```bash
nkp install kommander --init --airgapped > kommander.yaml
nkp install kommander \
  --installer-config kommander.yaml \
  --kubeconfig "${CLUSTER_NAME}.conf" \
  --kommander-applications-repository ./application-repositories/kommander-applications-v2.17.1.tar.gz
```

On Nutanix with `--self-managed`, Kommander install is part of cluster create. You still need the chart repository available or Flux cannot resolve `OCIRepository` objects.

### EKS Anywhere

```bash
eksctl anywhere download artifacts
tar -xvf eks-anywhere-downloads.tar.gz
eksctl anywhere download images -o images.tar
eksctl anywhere import images \
  -i images.tar \
  -r "${REGISTRY_MIRROR_URL}" \
  --bundles ./eks-anywhere-downloads/bundle-release.yaml
```

Run the download on amd64 when the cluster is amd64. The CLI pulls images for every provider and every supported Kubernetes version, so the tarball is large. Set `registryMirrorConfiguration` on the cluster spec before `create cluster`. For bare metal, also set `osImageURL` and `hookImagesURLPath` to URLs the machines can fetch inside the gap.

Curated packages are a second import. The cluster images tarball does not contain Harbor, Prometheus, and the rest of that catalog.

### RKE2

```bash
# on the connected host, for the exact version
curl -LO https://github.com/rancher/rke2/releases/download/v1.32.x%2Brke2r1/rke2-images.linux-amd64.tar.zst
curl -LO https://github.com/rancher/rke2/releases/download/v1.32.x%2Brke2r1/rke2.linux-amd64.tar.gz
curl -LO https://github.com/rancher/rke2/releases/download/v1.32.x%2Brke2r1/install.sh
```

On each node:

```bash
mkdir -p /var/lib/rancher/rke2/agent/images
cp rke2-images.linux-amd64.tar.zst /var/lib/rancher/rke2/agent/images/
INSTALL_RKE2_ARTIFACT_PATH=/root/rke2 ./install.sh
systemctl enable --now rke2-server
```

`/etc/rancher/rke2/config.yaml`:

```yaml
system-default-registry: registry.site.example:5000
cni: cilium
```

Use the tarball path for the first node and a registry for every node after that. Reloading a multi-gigabyte tarball onto every server is how air-gap upgrades time out.

## Air-gap upgrade

The upgrade command is the connected command. The prerequisite is the new bits in the mirror, plus the signature or the checksum the product checks.

### OpenShift

With OpenShift Update Service running inside the site:

```bash
POLICY_ENGINE_GRAPH_URI="$(oc -n openshift-update-service get updateservice service \
  -o jsonpath='{.status.policyEngineURI}/api/upgrades_info/v1/graph{"\n"}')"
oc patch clusterversion version --type merge \
  -p "{\"spec\":{\"upstream\":\"${POLICY_ENGINE_GRAPH_URI}\"}}"
oc adm upgrade
oc adm upgrade --to=4.19.6
```

Without the update service, after the release image and its signature ConfigMap are in the cluster:

```bash
oc adm upgrade --allow-explicit-upgrade \
  --to-image registry.site.example:5000/openshift/release@sha256:DIGEST
```

Apply the signature ConfigMap before the upgrade. CVO rejects a release image whose signature it cannot verify. Update OLM catalogs to the target minor before you move the payload, or installed operators lose their update path when the default catalog flips.

### NKP

Push the new Konvoy and Kommander bundles to the same registry. Import the new VM image into Prism and do not rename it; `nkp upgrade cluster nutanix` looks it up by name.

```bash
nkp upgrade kommander \
  --kommander-applications-repository ./application-repositories/kommander-applications-vX.Y.Z.tar.gz
nkp upgrade workspace "${WORKSPACE_NAME}"
nkp upgrade cluster nutanix \
  --cluster-name "${CLUSTER_NAME}" \
  --vm-image "${VM_IMAGE_NAME}"
```

Run the Kommander command from the extracted bundle root, or the relative chart paths miss.

### EKS Anywhere

Upgrade the CLI on the admin machine first, then repeat `download artifacts` / `download images` / `import images` with that CLI. Then:

```bash
eksctl anywhere upgrade plan cluster -f cluster.yaml
eksctl anywhere upgrade cluster -f cluster.yaml
```

Bare metal: point `osImageURL` and `hookImagesURLPath` at the new OS image before the upgrade, or the new machines have nothing to boot.

### GDC air-gapped

There is no automatic upgrade. List versions the management API already has, then request one:

```bash
kubectl --kubeconfig "${MANAGEMENT_API_SERVER}" \
  get userclustermetadata \
  -o custom-columns='TARGET_VERSION:.spec.kubernetesVersion'

kubectl --kubeconfig "${MANAGEMENT_API_SERVER}" apply -f - <<EOF
apiVersion: cluster.gdc.goog/v1
kind: UserClusterUpgradeRequest
metadata:
  name: ${CLUSTER_NAME}-to-target
  namespace: ${CLUSTER_NAMESPACE}
spec:
  clusterRef:
    name: ${CLUSTER_NAME}
  targetVersion: ${TARGET_VERSION}
EOF
```

`targetVersion` has to be one of the versions printed above. Asking for a version that was never imported does not reach out to the internet. It fails.

### RKE2

Stage the new tarball or mirror the new images, install the new version (`INSTALL_RKE2_VERSION` or the new artifact path), restart `rke2-server` on one server, wait until it is Ready and etcd is healthy, then the next server, then the agents. Do this inside a maintenance window. A kernel update from the OS mirror is a separate reboot.

## Failures that look like an upgrade bug and are an air-gap bug

| Symptom | Cause |
| --- | --- |
| `ImagePullBackOff` on a new operator or CSI pod, halfway through an upgrade | That image was not in the imageset or the bundle. The payload moved. The catalog did not. |
| CVO `ReleaseAccepted=False` with a signature error | The signature ConfigMap for the target digest was not applied, or the mirror rewrote the digest. |
| NKP Flux `OCIRepository` `x509: certificate signed by unknown authority` | Registry CA not in the trust bundle, or the bundle-based Kommander install left `flux-oci-mirror` without credentials. Registry-mirror installs hit this less often than `--bundle` installs. |
| Nodes join, then stay `NotReady` | CNI images were not mirrored. The kubelet is up. The CNI DaemonSet cannot pull. |
| EKS Anywhere preflight fails on a missing bundle manifest | `import images` was run with a bundle-release.yaml from a different CLI version than the one now running `upgrade cluster`. |
| GDC upgrade request stuck | `targetVersion` is not in `UserClusterMetadata`. Import the platform bundle that contains that version. |
| TLS errors only on some nodes | Those nodes do not trust the registry CA, or their clock is skewed. |
