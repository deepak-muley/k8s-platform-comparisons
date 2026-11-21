# Logging, monitoring, and platform apps

NKP is the one product here that upgrades its platform set because you upgraded NKP. The others either bake logging and monitoring into the platform release, version them as a separate add-on you must bump yourself, or leave the chart alone until you upgrade that chart.

Three piles, and they do not move together:

| Pile | What it is | Who bumps the version |
| --- | --- | --- |
| Platform observability | The logging and monitoring the product installed so the cluster can be operated | Sometimes the platform upgrade. Sometimes a separate add-on command. Sometimes you. |
| Platform apps | The default set the product considers part of the cluster (ingress, cert-manager, policy, the monitoring stack) | NKP upgrades this set as one step. Nobody else does. |
| Catalog apps | Optional apps you turned on from a catalog | You, one app at a time, on every product including NKP. |

A Helm release the product does not own never moves because the Kubernetes version moved.

## What moves when the platform moves

| | Logging and monitoring the product installed | The rest of the default platform set | Catalog / optional apps | Your own Helm chart |
| --- | --- | --- | --- | --- |
| NKP | In the platform set. `kube-prometheus-stack`, the logging stack, and the other default `AppDeployment`s bump together, before Kubernetes. | Same command. cert-manager, Traefik, Flux, gatekeeper, and the rest of that release's list. | Not in the set. Upgrade the one `AppDeployment` when the catalog collection has a new version. | Stays. Kommander will not touch it. |
| OpenShift | Monitoring is a payload operator. It moves with `oc adm upgrade`. You do not version it. Logging is an OperatorHub subscription. It does not move with the payload unless that subscription is `Automatic` and the new catalog index has a CSV. | Console, ingress, auth, image registry, and the monitoring operator are the same payload. They move, and you cannot skip one. | GitOps, Service Mesh, ODF, and the rest of OperatorHub. Upgrade the operator before the cluster minor, or approve the InstallPlan. `oc adm upgrade` does not walk a list of them. | Stays. |
| GKE | Cloud Logging and Cloud Monitoring collectors are on the node. They move when the node image moves. Managed Prometheus is on by default from 1.27 and Google versions it. There is no chart version to pin. | There is no Kommander-style set. Add-ons you enabled (Config Connector, Backup for GKE, Filestore CSI) are Google-managed and move with GKE when enabled. | Marketplace charts you installed stay. | Stays. |
| EKS | Not installed unless you add `amazon-cloudwatch-observability` or `adot`. Those are managed add-ons. `update-cluster-version` does not bump them. | No default set. `vpc-cni`, `coredns`, and `kube-proxy` are add-ons with the same rule: they stay until `update-addon`. | No catalog that rides the cluster version. | Stays. |
| EKS Anywhere | Not installed. A curated package such as Prometheus is a `Package` object. `upgrade cluster` does not change its version. | No default set beyond Cilium, which is part of the cluster upgrade. | Each package has its own version. Bump that object. Air gap: import that package's images separately. | Stays. |
| AKS | Azure Monitor and managed Prometheus are Microsoft-run. You enable them. You do not pick the collector chart, and `az aks upgrade` is not an "upgrade all platform apps" step. The collector moves when Microsoft rolls it, often with the node image. | No default set. Azure Policy and the HTTP routing add-on, if you enabled them, are the same kind of Microsoft-owned add-on. | No catalog tied to the Kubernetes version. | Stays. |
| RKE2 | Nothing, until you install it. | RKE2's own manifests (CNI, CoreDNS, and the other charts in the release) move when the RKE2 version moves. That is not monitoring. | — | Stays. |
| Rancher | `rancher-monitoring` and `rancher-logging` are charts you installed from Apps & Marketplace. Upgrading the downstream Kubernetes version does not upgrade them. Upgrading Rancher does not upgrade them either. | Before a Rancher minor, move each feature chart to the latest patch of its current major. After the Rancher minor, move the chart onto the new major yourself. The chart major tracks the Rancher minor. | Partner charts are the same: Installed Apps, one upgrade at a time. | Stays, unless Fleet's Git repo bumped the chart. |

## NKP

`--self-managed` installs Kommander and the default platform apps. The list for a release is whatever that Kommander version enables. Read it after install, do not memorize it:

```bash
kubectl get appdeployments -A
kubectl get helmreleases -n kommander
```

A 2.18 workspace upgrade prints the pairs it changes (`kube-prometheus-stack-78.4.0` to `kube-prometheus-stack-82.13.6`, `cert-manager`, `gatekeeper`, `kommander-flux`, and the rest). That is the set. It runs before any `nkp upgrade cluster`, because those chart versions do not work on the Kubernetes version in the next NKP release.

Through 2.18:

```bash
nkp upgrade kommander
# air gap, from the extracted bundle root
nkp upgrade kommander \
  --kommander-applications-repository ./application-repositories/kommander-applications-vX.Y.Z.tar.gz

nkp get workspaces
nkp upgrade workspace "$WORKSPACE_NAME"
```

`nkp upgrade kommander` moves the management cluster. `nkp upgrade workspace` moves every cluster in that workspace. Split prod and nonprod into different workspaces or the platform-app bump is the whole workspace.

On 2.19, `nkp upgrade workspace` is deprecated. Non-ClusterClass clusters use `nkp upgrade platform-apps`. ClusterClass clusters use the per-cluster upgrade, which still runs platform apps before Kubernetes. The management cluster is on 2.19 first. GitRepository-based catalogs are no longer supported. Migrate those `AppDeployment`s to an OCI catalog before the upgrade. The `--kommander-applications-repository` flag is gone on 2.19.

You do not `helm upgrade` a platform `HelmRelease` Kommander owns. The next platform upgrade puts the chart back to the version in the release. Change values through the `AppDeployment` config override.

Catalog apps are the other list. Register a collection, then enable one app:

```bash
nkp create catalog-collection <collection>
nkp create appdeployment <app> --workspace "$WORKSPACE_NAME" --config-overrides <configmap>
```

A new collection version does not roll every enabled app. Upgrade that `AppDeployment` from the UI or the CLI when you want that app's new chart. Harbor does not move because Prometheus did.

A logging major inside the platform set can be a migration, not a chart bump. The 2.18 Kommander upgrade checks a Loki 2 to Loki 3 move before it continues. Custom log storage has to be ready before that step. The platform upgrade will not skip logging and do the rest.

Watch:

```bash
kubectl get helmreleases -A
kubectl get appdeployments -A
kubectl get nkpcluster -n kommander host-cluster \
  -o jsonpath='{range .status.conditions[?(@.status=="False")]}{.type}: {.message}{"\n"}{end}'
```

A 60-minute timeout from `nkp upgrade kommander` means the CLI stopped waiting. The controllers keep reconciling. Read the conditions before you run it again.

Air gap: the Kommander image bundle and `kommander-applications-*.tar.gz` are the platform set. `nkp-catalog-applications-image-bundle-*.tar` is the optional catalog. Push that bundle only if you enable catalog apps. A cluster upgrade with no catalog bundle still upgrades the platform set, and catalog `AppDeployment`s then have nothing to pull.

## OpenShift

Monitoring is already installed. It is the `monitoring` ClusterOperator. `oc adm upgrade` moves Prometheus, Alertmanager, and the node exporter to the build in that payload. A CVE in that stack is a z-stream of the whole cluster, not a monitoring-only release.

Turn on user-workload metrics without changing the version:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
```

That rolls monitoring pods. It does not roll nodes unless you also change a retention or a node selector that the operator renders as a MachineConfig.

Logging is not in the payload. Subscribe to it from OperatorHub (`loki-operator`, and the cluster logging or cluster observability operator for the minor you run). The object is a `Subscription`. The version moves when you approve an `InstallPlan`, or immediately if `installPlanApproval` is `Automatic` and the catalog index has a newer CSV.

```bash
oc get subscription -A
oc get installplan -A
oc get csv -A
oc get co monitoring
```

Do this before `oc adm upgrade` to the next minor. The payload upgrade flips the default catalog index from `v4.18` to `v4.19`. An operator with no CSV in the new index sits at `UpgradePending`. Automatic approval does not save you if the new index has no path. Mirror those operator packages in the same `oc-mirror` run as the payload.

Console, ingress, authentication, and the image registry are payload operators. They move with monitoring. You cannot hold the console back and take the rest.

GitOps, Service Mesh, and OpenShift Data Foundation are catalog operators, same rule as logging. One InstallPlan each. There is no command that means "upgrade every OperatorHub install to match this OpenShift minor."

## GKE

Logging and monitoring collectors are not charts. Select what is collected. Google ships the agent on the node and moves it when the node pool moves.

```bash
gcloud container clusters update "$CLUSTER" --location "$REGION" \
  --logging=SYSTEM,WORKLOAD \
  --monitoring=SYSTEM
```

`--logging=NONE` turns workload and system log collection off. The flag replaces the set. Component names beyond `SYSTEM` and `WORKLOAD` are in `gcloud container clusters update --help` for the gcloud you have.

Managed Prometheus is on for new clusters at 1.27 and later. `--no-enable-managed-prometheus` turns the collector off. You cannot pin the collector image. A CVE in it arrives with a GKE patch, which is a control-plane or node upgrade, not a Helm upgrade.

There is no platform-app step in a GKE upgrade. After the node pools finish, the Google collectors are the new build. kube-prometheus-stack you installed is the old build until you upgrade that release.

Config Connector, Backup for GKE, and the Filestore CSI driver are add-ons. Enable one with `--update-addons`. Google picks the version. Do not pass `--addons` on create to flip one of them. That flag replaces the default set.

## EKS

A new cluster has no Prometheus and no log agent. The AWS-supported observability install is an add-on:

```bash
aws eks create-addon --cluster-name "$CLUSTER" \
  --addon-name amazon-cloudwatch-observability --addon-version "$V"
aws eks create-addon --cluster-name "$CLUSTER" \
  --addon-name adot --addon-version "$V"

aws eks describe-addon-versions \
  --addon-name amazon-cloudwatch-observability --kubernetes-version 1.32
```

Patch that add-on without touching the control plane or the AMI:

```bash
aws eks update-addon --cluster-name "$CLUSTER" \
  --addon-name amazon-cloudwatch-observability \
  --addon-version "$V" --resolve-conflicts PRESERVE
```

`update-cluster-version` leaves this add-on where it is. A Kubernetes minor whose compatibility table drops the old add-on build fails later, at the next `update-addon`, not during the control-plane upgrade. Put the observability add-on in the same pipeline stage as `vpc-cni`.

kube-prometheus-stack, Loki, or Grafana you installed with Helm stay on the chart version you installed. EKS has no catalog pass that rewrites them.

## EKS Anywhere

Curated packages are the catalog. Prometheus and Harbor are packages, not part of `upgrade cluster`.

```bash
kubectl get packages -A
```

Change `spec.packageVersion` on that `Package` (`packages.eks.amazonaws.com/v1alpha1`). The package controller reconciles it. A cluster upgrade that replaces machines does not change the field. If the package's Kubernetes range does not include the new minor, the package breaks after the machines roll and you then bump the package. Do that bump on the canary before the prod roll.

Air gap: `images.tar` from `download images` does not contain curated package images. Import them in a second step or the Package stays `ImagePullBackOff` after a perfectly successful cluster upgrade.

## AKS

Managed Prometheus is an enablement, not a chart pin:

```bash
az aks update -g "$RG" -n "$CLUSTER" --enable-azure-monitor-metrics \
  --azure-monitor-workspace-resource-id "$WORKSPACE_ID"
```

The older Container Insights path is the `monitoring` add-on. Prefer managed Prometheus for a new cluster. Microsoft runs the collector and Azure Managed Grafana. You do not `helm upgrade` them, and you do not get an NKP-style step that bumps them to a known chart version before `az aks upgrade`. A collector CVE shows up as a Microsoft rollout or a new node image.

`--node-image-only` can move the node-side agent without moving Kubernetes. It does not move a Helm chart you installed, and it is not a substitute for a chart that calls a removed API.

## Rancher and RKE2

RKE2 does not install monitoring or logging. Restarting `rke2-server` upgrades CoreDNS and the CNI chart that shipped in that RKE2 version. It does not upgrade a monitoring chart.

Rancher's feature charts (`rancher-monitoring`, `rancher-logging`, and the others under Apps & Marketplace) are Helm releases on the downstream cluster. The chart major tracks the Rancher minor (`100.x` for one Rancher minor, `101.x` for the next). The `+up` suffix is the upstream chart that build tracks.

Upgrade order Rancher documents:

1. Latest patch of the current Rancher minor.
2. Latest patch of each installed feature chart, still on its current major.
3. Next Rancher minor.
4. Each feature chart, onto the major that matches the new Rancher minor. One Installed App at a time.

Skipping step 2 or 4 is how monitoring breaks across a Rancher upgrade. Skipping them is also allowed: a downstream Kubernetes upgrade does not refuse to start because the monitoring chart is old. The chart keeps running until an API it calls is gone. That is the opposite of NKP, which will not take the new Kubernetes until the platform set has moved.

Fleet is how you stop doing step 4 by hand. A `GitRepo` that pins the chart version bumps every target cluster when that git revision moves. Put the chart bump in a different git change from the RKE2 version bump so a bad chart does not ride a node roll.

## Patch one app, or the whole set

| I need to patch | NKP | OpenShift | GKE | EKS | AKS | Rancher |
| --- | --- | --- | --- | --- | --- | --- |
| Prometheus only, product-owned | You don't. The next platform upgrade moves the whole set. A catalog Prometheus is one `AppDeployment`. | You don't, if it is the payload `monitoring` operator. A z-stream moves the cluster. | You don't. The next node or control-plane patch moves the collector. | `update-addon` if it is the CloudWatch or ADOT add-on. Helm if it is yours. | You don't. Microsoft rolls the managed collector. | Upgrade the `rancher-monitoring` chart. Nothing else moves. |
| Logging only, product-owned | Same, it is in the platform set. A Loki major can be its own migration inside that upgrade. | Approve the logging operator's InstallPlan. Monitoring does not move. | Same as Prometheus. `--logging` only changes what is collected. | Same as Prometheus. | Same as Prometheus. | Upgrade the `rancher-logging` chart. |
| One catalog app | That `AppDeployment`. | That operator's InstallPlan. | That Marketplace release, yourself. | — | — | That Installed App. |
| Every platform app, because Kubernetes is about to move | `nkp upgrade workspace` or `nkp upgrade platform-apps`, first. | Payload apps move with `oc adm upgrade`. OperatorHub apps you upgrade first, one subscription at a time. | No step. Collectors move with the nodes. Your charts you upgrade yourself, before the minor if they call removed APIs. | No step. `update-addon` per add-on. Your charts yourself. | No step. | Each feature chart, yourself, around the Rancher upgrade. Not around the downstream Kubernetes upgrade unless the chart's kube-version says so. |
