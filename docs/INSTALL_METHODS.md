# dellctl, Helm or Operator: which one, and when

Three tools show up in the install. They are not three competing installers: **dellctl generates, Helm and the
Operator install**. This page says what each does, which one this repo uses on each platform, and why.

## 1. What each one is

| | dellctl | Helm (`container-storage-modules` chart) | CSM Operator |
|---|---|---|---|
| Role | CLI that builds the configuration (values file, secrets, StorageClasses, replication wiring) from one YAML file | Package manager that installs the driver and the replication controller from the chart | Kubernetes operator: you create a `ContainerStorageModule` resource and the Operator deploys and reconciles the driver |
| Input | `--from-file` YAML (storage, replication, prefix, RPO) | A values file | A `ContainerStorageModule` custom resource + an `isilon-creds` secret |
| Output | Files (with `output: true`) or an install | A Helm release (`isilon`) | A `ContainerStorageModule` in state `Succeeded` |
| Day-2 | Re-run offline to compare against your edited copy | `helm upgrade`, `helm rollback`, history | Change the CR; Operator upgrades are approved in OLM |
| Needs a cluster during use | Optional (`output` mode generates offline) | Yes | Yes (OLM on OpenShift) |

Dell's CLI reference documents `--output` ("dumps every generated yaml to a file and does not execute an
installation"), `--operator-install` (installs the CSM CRDs) and says replication is not available through CLI
flags: it must be configured through the `--from-file` parameters. The v1.18.2 `dellctl` binary contains
`dellctl install powerscale --from-file=config.yaml` and the array password prompt this repo answers.

## 2. What this repo does

| Platform | Install | Why |
|---|---|---|
| **K3s** (`-i inventory/k3s`) | **dellctl generates, Helm installs** | The lab-verified path of the guide (Part B). dellctl produces the values, the replication StorageClass and the cross-cluster Secret/ConfigMap; Helm installs the driver and controller and gives revision history and rollback |
| **OpenShift** (`-i inventory/openshift`) | **CSM Operator** (certified, OperatorHub, Manual approval) + `repctl` | Part C of the guide. The Operator is the Red Hat-certified route, OpenShift already ships the snapshot CRDs and controller, and OLM handles lifecycle. Replication is wired with `repctl cluster inject`, StorageClasses are created explicitly |

Per cluster pair use **one** wiring method for replication: dellctl-generated Secrets (K3s) **or** `repctl inject`
(OpenShift). Mixing them creates conflicting remote-cluster Secrets. The `csm_k3s` and `csm_ocp` roles assert the
platform to keep them apart.

## 3. Decision guide

| Situation | Use | Notes |
|---|---|---|
| Upstream Kubernetes or K3s, you want revision history and simple rollback | dellctl (generate) + Helm | Add the snapshot CRDs and controller yourself (the generated manifest has them) |
| OpenShift | Operator | Keep the Operator approval on Manual so it cannot move to a version outside the support matrix |
| Air-gapped OpenShift | Operator, mirrored with `oc-mirror` | See the guide section O1 |
| You want the cluster to reconcile the driver continuously (GitOps with a CR) | Operator | Works on upstream Kubernetes too; install the Operator first |
| One-off lab, no Helm, no OLM | dellctl direct install | Not used here: it gives you no values file to keep under change control |
| You already run the older CSI Operator | Migrate to the CSM Operator | Dell documents a CSI to CSM Operator migration; it is non-disruptive for mounted volumes but you cannot create volumes or snapshots meanwhile |

## 4. Trade-offs

- **dellctl + Helm**: transparent and scriptable, and everything you changed is in a values file. The cost is more
  moving parts for replication: you split the generated manifest, apply the prerequisites, install with Helm, then
  re-apply `06_replication.yaml` and restart the controller after any `helm upgrade` (the chart can reset the
  ConfigMap).
- **Operator**: fewer manual steps and managed lifecycle. The cost is OLM and the CR schema: field names vary by CSM
  version (check the sample CR for your release), and upgrades are an approval flow instead of a command.
- **dellctl alone** (without Helm or the Operator) has no day-2 story of its own; treat it as a configuration
  generator.

## 5. What is not proven

- Dell's docs I read offer both Operator and Helm for PowerScale on Kubernetes and do not state a preference; the
  split above follows the guide, not a Dell recommendation.
- `dellctl install powerscale` is not described in Dell's public CLI reference (it lists PowerStore). Its presence
  and password prompt were confirmed from the binary only. Confirm with `dellctl install powerscale --help` on your
  build before relying on any flag.
- Nothing here was run end to end: the guide marks Parts C and D as not lab-run.

## Sources

- Dell CSM docs, CLI reference (`dellctl install`): `content/docs/tooling/cli` in `dell/csm-docs`
- Dell CSM docs, PowerScale installation (Operator and Helm cards) and CSI to CSM Operator migration:
  `content/docs/getting-started/installation/` in `dell/csm-docs`
- The guide: `docs/PowerScale CSM Replication ... (v5).html`, sections K3-K6, O1-O6
