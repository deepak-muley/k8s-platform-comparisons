# Amazon EKS

EKS in a region is a managed control plane plus node groups you own (or Auto Mode, where AWS owns the nodes too). The part that matters for this repo: VPC CNI, CoreDNS, kube-proxy, and the EBS CSI driver are managed add-ons with their own versions. They do not move when you upgrade Kubernetes. That is both the fastest CSI/CNI path in this comparison and the easiest one to forget.

EKS is not an air-gapped product. A private-endpoint cluster still uses AWS APIs in the region. The air-gap product is [EKS Anywhere](eks-anywhere.md).

The control plane is the part Cluster API cannot do: `update-cluster-version` moves Kubernetes with no instance you replace. Worker nodes are still a replacement, and the EKS-optimized AMI is built for one Kubernetes version, so a node Kubernetes bump is a new AMI. An OS-only week is a new AMI at the same Kubernetes version, then `update-nodegroup-version` with no cluster-version change. That split (control plane independent, nodes image-coupled, OS-only still a new instance) is the useful contrast with CAPX, where both the control plane and the workers are image replacements. Details are in [Cluster API](../capi.md).

## Core vs add-on

| | Examples | Version source | Moves when | Speed |
| --- | --- | --- | --- | --- |
| Core | API server, etcd, scheduler, controller-manager | `aws eks update-cluster-version` | You, one minor at a time, inside the versions `describe-addon-versions` and the cluster API currently offer | API operation. No node reboot. |
| Core, your nodes | EKS-optimized AMI and the kubelet on a managed node group, or a Karpenter `NodeClass` | The node group launch template / AMI, or Karpenter drift | `update-nodegroup-version`, or a drifted NodeClass | 3 |
| Managed add-on, platform core in practice | `vpc-cni`, `kube-proxy`, `coredns` | `aws eks describe-addon-versions --addon-name vpc-cni --kubernetes-version 1.32` | `aws eks update-addon`. Not during `update-cluster-version`. | 2. DaemonSet or Deployment roll. |
| Managed add-on | `aws-ebs-csi-driver`, `aws-efs-csi-driver`, `snapshot-controller`, `eks-pod-identity-agent`, ADOT | Same API, per add-on | `update-addon` | 2 |
| Add-on you own | Any other CSI, any Helm chart, a CNI you insisted on running beside VPC CNI | Your chart | `helm upgrade` | 2 |

Self-managed add-ons (you applied the upstream VPC CNI manifest yourself) do not appear in `aws eks list-addons`. They also do not get `configurationValues` or a clean `PRESERVE` story. Prefer the managed add-on unless you already have a reason not to.

VPC CNI is the supported primary CNI. Prefix delegation, security groups for pods, and custom networking are VPC CNI features. Replacing it with Cilium as the only CNI is outside the normal EKS design. If a team needs Cilium policy, read the current AWS guidance for that minor before you install a second data plane. Two CNIs is a production incident waiting on a node restart.

## Create

```bash
aws eks create-cluster \
  --name "${CLUSTER}" \
  --kubernetes-version 1.32 \
  --role-arn "${CLUSTER_ROLE_ARN}" \
  --resources-vpc-config subnetIds="${SUBNETS}",endpointPublicAccess=true,endpointPrivateAccess=true

aws eks wait cluster-active --name "${CLUSTER}"

aws eks create-nodegroup \
  --cluster-name "${CLUSTER}" \
  --nodegroup-name general \
  --subnets ${SUBNETS} \
  --node-role "${NODE_ROLE_ARN}" \
  --scaling-config minSize=3,maxSize=10,desiredSize=3 \
  --update-config maxUnavailable=1

aws eks create-addon \
  --cluster-name "${CLUSTER}" \
  --addon-name vpc-cni \
  --addon-version "${VPC_CNI_VERSION}" \
  --service-account-role-arn "${VPC_CNI_ROLE_ARN}" \
  --configuration-values '{"env":{"ENABLE_PREFIX_DELEGATION":"true"}}'

aws eks create-addon \
  --cluster-name "${CLUSTER}" \
  --addon-name aws-ebs-csi-driver \
  --addon-version "${EBS_CSI_VERSION}" \
  --service-account-role-arn "${EBS_CSI_ROLE_ARN}"

aws eks create-addon --cluster-name "${CLUSTER}" --addon-name coredns --addon-version "${COREDNS_VERSION}"
aws eks create-addon --cluster-name "${CLUSTER}" --addon-name kube-proxy --addon-version "${KUBE_PROXY_VERSION}"

aws eks update-kubeconfig --name "${CLUSTER}"
kubectl get ds -n kube-system aws-node
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
kubectl get storageclass
```

The EBS CSI add-on does nothing useful until its IAM role can call `ec2:CreateVolume` and the related calls. Create the role before the add-on. Pod identity or IRSA both work; pick the one the rest of the cluster uses and do not mix them on the same service account.

Find a legal add-on version instead of guessing:

```bash
aws eks describe-addon-versions \
  --addon-name aws-ebs-csi-driver \
  --kubernetes-version 1.32 \
  --query 'addons[0].addonVersions[].addonVersion'
```

The schema for `configuration-values` is per version:

```bash
aws eks describe-addon-configuration \
  --addon-name vpc-cni \
  --addon-version v1.20.3-eksbuild.1
```

Use the version you are about to install. The schema changes between builds.

EKS Auto Mode is the other create path: AWS provisions and upgrades the nodes, and a set of add-ons is part of that mode. Use it when you want the GKE-Autopilot shape. Use managed node groups or Karpenter when you need to control the AMI, the CNI config, or a DaemonSet AWS will not run for you. The add-on commands in the rest of this page are the managed-node-group path.

## Day-2 config of an add-on

Change prefix delegation, an env var, or a resource limit through the add-on API so the next upgrade knows about it.

```bash
aws eks update-addon \
  --cluster-name "${CLUSTER}" \
  --addon-name vpc-cni \
  --addon-version "${VPC_CNI_VERSION}" \
  --resolve-conflicts PRESERVE \
  --configuration-values '{"env":{"ENABLE_PREFIX_DELEGATION":"true","WARM_PREFIX_TARGET":"1"}}'
```

`configuration-values` replaces the configuration blob. Include every key you still want. A later call that passes only one env var resets the others to the add-on defaults.

`resolve-conflicts` is about edits someone made with `kubectl`, not about the blob:

| Value | Behavior |
| --- | --- |
| `PRESERVE` | Keep kubectl edits on fields the add-on also manages. Test that on a non-prod cluster. A preserved field can be illegal on the new add-on version. |
| `OVERWRITE` | Put those fields back to the EKS default. |
| `NONE` | Fail the update if a field conflicts. |

Prefer putting the change in `configuration-values` and using `PRESERVE` only for the leftovers. The add-on writes to the cluster on create, update, and delete. It does not reconcile every few minutes the way an older EKS add-on controller did. A kubectl edit will sit there until the next `update-addon`, and then `PRESERVE` or `OVERWRITE` decides.

Watch the roll:

```bash
aws eks describe-addon --cluster-name "${CLUSTER}" --addon-name vpc-cni \
  --query 'addon.{status:status,version:addonVersion,health:health}'
kubectl rollout status ds/aws-node -n kube-system
```

A VPC CNI roll restarts `aws-node` on every node. Existing established flows survive a normal restart. New pods on a node that is mid-roll can fail to get an IP for a few seconds. Do it in a window, one cluster at a time, even though it is "only" speed 2.

## Upgrade Kubernetes

```bash
aws eks list-insights --cluster-name "${CLUSTER}"
aws eks update-cluster-version --name "${CLUSTER}" --kubernetes-version 1.33
aws eks describe-update --name "${CLUSTER}" --update-id "${UPDATE_ID}"
aws eks wait cluster-active --name "${CLUSTER}"
```

Then the nodes. Managed node group, same Kubernetes version as the control plane, new AMI:

```bash
aws eks update-nodegroup-version \
  --cluster-name "${CLUSTER}" \
  --nodegroup-name general \
  --update-config maxUnavailable=1
```

Karpenter: set the new AMI or the new Kubernetes version on the `EC2NodeClass` / `NodePool` and let drift replace nodes. `maxUnavailable` on the node group has an analogue in the NodePool disruption budget. Set it. The default disruption behavior is how a Karpenter cluster replaces more nodes than the workload can lose.

Then the add-ons, each at a version `describe-addon-versions` says is valid on the new minor:

```bash
for addon in vpc-cni kube-proxy coredns aws-ebs-csi-driver; do
  aws eks update-addon \
    --cluster-name "${CLUSTER}" \
    --addon-name "${addon}" \
    --addon-version "${VERSION_FOR[$addon]}" \
    --resolve-conflicts PRESERVE
done
```

If you skip this loop, the cluster minor is new and the CNI is old. That works until it does not, usually at the next minor, when the old add-on version disappears from the compatibility table. The proof for a target minor is `aws eks describe-addon-versions --kubernetes-version` that minor, and a hit in `compatibilities[].clusterVersion`. The same check for the other products is in [Compatibility](../compatibility.md).

### Node AMI only

`update-nodegroup-version` without a cluster-version change, when AWS has published a new EKS-optimized AMI for the same Kubernetes minor, is the CVE path. CoreDNS and VPC CNI do not move. That is what you want for a weekly image roll.

## CNI

Day-to-day knobs that are real on EKS:

- `ENABLE_PREFIX_DELEGATION` and the warm-prefix / warm-IP env vars, via `configuration-values`. This is how a node gets more pod IPs than the instance's ENI limit. It changes IP allocation. Test it before you turn it on fleet-wide. Existing nodes pick it up as `aws-node` rolls.
- Custom networking (pods in a different subnet than the node) and security groups for pods. Both are VPC CNI features with their own IAM and subnet requirements. They are create-or-migrate projects, not an env var you flip at noon.
- `NetworkPolicy` with the VPC CNI network policy agent, if you enabled it on that add-on version.

There is no separate "upgrade Cilium" because the primary CNI is not Cilium. The upgrade unit is the `vpc-cni` add-on version, for example `v1.20.3-eksbuild.1`. Build numbers (`eksbuild.N`) are AWS's patch on that upstream version. You can move the build the same day, speed 2.

## CSI

EBS CSI is the add-on `aws-ebs-csi-driver`. EFS is `aws-efs-csi-driver`. They upgrade like VPC CNI, and they have their own IAM roles.

```bash
aws eks update-addon \
  --cluster-name "${CLUSTER}" \
  --addon-name aws-ebs-csi-driver \
  --addon-version "${EBS_CSI_VERSION}" \
  --service-account-role-arn "${EBS_CSI_ROLE_ARN}" \
  --resolve-conflicts PRESERVE
```

A StorageClass with `provisioner: ebs.csi.aws.com` is speed 1. `gp3` is the type you want in the parameters. The default class EKS creates (`gp2`) is the one to stop using for new volumes. Changing the default annotation does not convert existing PVs.

In-tree `kubernetes.io/aws-ebs` is gone. Old manifests still in a Git repo fail at PVC create, which is a better failure than a silent in-tree mount.

After a CSI roll, test a new volume, a remount of an old volume on a different node, and an expansion. EBS volumes are AZ-scoped. A pod that lands in another AZ stays `Pending` with a volume affinity error. That is not a CSI regression. Topology is doing its job. `volumeBindingMode: WaitForFirstConsumer` is how the scheduler and the driver agree on the AZ.

## At scale

There is no Fleet Manager equivalent. A pipeline per account, roughly:

1. `list-insights` and stop on a high-severity insight.
2. `update-cluster-version`. Concurrency of control-plane updates can be higher than node updates.
3. Wait `cluster-active`.
4. `update-addon` for vpc-cni, kube-proxy, coredns, aws-ebs-csi-driver, each pinned to a version you already ran on the canary cluster.
5. `update-nodegroup-version` with `maxUnavailable=1`, or Karpenter drift with a tight disruption budget.
6. A smoke test that creates a PVC and a pod.

Do step 4 even though it is tempting to batch it later. Record `addonVersion` per cluster. Drift here is invisible until a minor upgrade rejects the old build.

EKS extended support keeps a minor alive longer at a cost. It does not upgrade anything for you. It buys calendar time. The add-on compatibility table still applies.

## Traps

- Assuming `update-cluster-version` updated VPC CNI. It did not.
- `configuration-values` that contains only the one key you are changing. Everything else in the blob resets.
- `PRESERVE` on a field the new add-on version removed. The update succeeds and the DaemonSet is wrong. Read the add-on health, not just the API status.
- Forgetting the CSI driver's IAM role when moving from IRSA to pod identity. The driver pods are Running and every CreateVolume returns 403.
- Surge / `maxUnavailable` larger than the PDB allows. The node group update sits in `Updating` and the drain events show PDB violations.
- A private cluster with no VPC endpoints for ECR, EC2, and STS. Nodes come up and cannot pull `aws-node` or talk to the EC2 API. That looks like a CNI bug.
