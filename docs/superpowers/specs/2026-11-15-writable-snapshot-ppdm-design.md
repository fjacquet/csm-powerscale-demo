# PowerScale writable snapshots behind PPDM backups — design

Date: 2026-11-15. Status: draft, awaiting approval. Scope: new Ansible role + playbook + docs, K3s and OpenShift tracks.

## Intent and why

Dell states that PowerProtect Data Manager (PPDM) can back up important PVs through PowerScale writable snapshots.
No document we hold says so in those words. The goal is an experiment, repeatable on a lab, that either **observes**
a OneFS writable snapshot during a PPDM backup of a PV, or records that it did not happen. Whatever the outcome, it is
evidence for the customer demo and for a precise question back to Dell.

## What the sources say (all local, `docs/references`, `docs/openapi`)

- CSM 1.18 admin guide, ch. 15 "PowerScale Writable Snapshots": a PVC with `dataSource: VolumeSnapshot` on a StorageClass
  with `csi-isilon.dellemc.com/writable-from-snapshot: "true"` becomes a OneFS writable snapshot
  (`POST /platform/14/snapshot/writable`, COW, SmartQuota hard limit = PVC size). Needs OneFS >= 9.13, K8s >= 1.34,
  driver >= 2.18.0, `snapshot.enabled`, `resizer.enabled`. Limits: same AccessZone as the source snapshot, no
  cross-namespace snapshot, source snapshot undeletable while dependents exist (`SnapshotHasDependents`),
  `reclaimPolicy: Delete` or the OneFS directory is orphaned.
- PPDM 20.3 Kubernetes guide: non-FCD backups create a CSI snapshot, then a **temporary PVC** from it, read by the cProxy
  data mover. ConfigMap `ppdm-snapshot-storage-class-mapping` (ns `powerprotect`, `<SC of backed-up PVC>: <SC for
  temporary PVC>`) chooses that PVC's StorageClass. `ppdm-snapshot-class-mapping` chooses the VolumeSnapshotClass.
  `ppdm-custom-namespace-storage-class-mapping` chooses the data-mover namespace per PowerScale StorageClass. Velero/OADP
  carries metadata only. The word "writable" appears nowhere in the PPDM guide.
- OneFS CLI guide: `isi snapshot writable list|view|create|delete`. OpenAPI 9.13: `GET /platform/14/snapshot/writable`.
- OpenAPI PPDM 20.3: `/api/v2/login`, `/api/v2/kubernetes-clusters`, `/api/v2/protection-rules`,
  `/api/v3/protection-policies`, `/api/v2/activities`.

The join (PPDM temporary PVC -> writable StorageClass -> OneFS writable snapshot) is **inferred**, not documented.

## Goals

- Preflight that fails early on version/feature gaps (OneFS, K8s, driver, sidecars, snapshot controller, PPDM 20.3).
- CSM layer, no PPDM needed: StorageClass B (writable) + VolumeSnapshotClass; snapshot `pgdata`; PVC from it; assert
  a writable snapshot exists on the array; clean up in the right order.
- PPDM layer: render the PPDM ConfigMaps (A -> B mapping), and optionally drive PPDM through its REST API
  (login, register cluster, policy on the `pgdata` namespace, start, follow activity).
- Evidence capture: during the PPDM job, poll the array for writable snapshots and the cluster for
  `volumesnapshot,pvc`; write a timeline file. The run ends with an explicit result: OBSERVED / NOT OBSERVED.
- Both tracks. K3s is not mentioned in the PPDM 20.3 guide: it is tried anyway and recorded as an experiment.
- Docs: new part in `docs/USER_GUIDE.md`, role README, and a demo walkthrough (next section).

## Demo walkthrough documentation (required)

A separate page, `docs/WRITABLE_SNAPSHOT_DEMO.md`, linked from `docs/USER_GUIDE.md`, written for the presenter. Visuals
are Mermaid diagrams (rendered by GitHub), never ASCII-art tables or boxes. It contains:

- **State diagram** (`stateDiagram-v2`) of one run: Preflight -> CSM check -> PPDM configured -> Backup running ->
  Evidence collected -> OBSERVED or NOT OBSERVED, with the failure state of each step and the command that tells which
  state you are in.
- **Sequence diagram** (`sequenceDiagram`) of the inferred chain: PPDM controller, CSI snapshotter, csi-isilon driver,
  OneFS API, cProxy data mover, with the snapshot, the temporary PVC and the writable snapshot as messages. Steps that
  are inferred rather than documented are marked as such in a note.
- **Flowchart** (`flowchart`) of the decision when the result is NOT OBSERVED: which risk to check next (namespace,
  cleanup order, backup path, platform support), pointing to the risk list.
- Per step: what to run, what the audience should see, expected output, what to say. Plain Markdown lists, no ASCII tables.
- **Generated evidence:** `evidence.yml` also writes the observed timeline of the real run as a Mermaid `sequenceDiagram`
  file (event, time, source: cluster or array), so the presenter can show what actually happened, not a generic picture.
- Builder check: Mermaid blocks are validated with `mmdc` if installed, otherwise reviewed by eye. Rendered SVG exports
  for slides are optional and out of scope for the first version.

## Non-goals

- Installing or sizing PPDM, Data Domain, or the lab. No Velero/OADP tuning. No VolumeGroupSnapshot (block arrays only).
- No restore demo, no change to DR playbooks, replication, `app-b`/`app-c`.
- No dependency on the bottkars repo.

## Anti-goals

- `site.yml` reaching the new playbook, or any DR guard weakened.
- Declaring success without observing a writable snapshot on the array during the backup.
- Secrets (PPDM password, array credentials) in files or logs; per-track data outside
  `inventory/<t>/group_vars/all/platform.yml`.
- Leaving OneFS writable directories behind (cleanup order, `reclaimPolicy: Delete`).

## Constraints

- Repo rules: playbook at the repo root; role-prefixed variables (`demo_wsnap_`); Jinja only at the end of task names;
  rendered manifests applied with `kubectl apply` like `demo_wiki`; `ps_access: api|ssh` supported for array checks.
- StorageClass parameters are immutable: B is created once with final values; A is the existing `dr_protect_sc`, untouched.
- B must copy `IsiPath` and `AccessZone` of A (same-zone rule) and must not carry replication parameters.
- Lab is not public-compatible with E-Lab: version support for PPDM 20.3 x OCP/K3s x K8s 1.34 cannot be checked
  here; preflight checks what it can, the rest is an experiment.
- Nothing has run against real clusters; validation here is lint, syntax-check, template golden test.

## Approaches considered

- **1 (chosen): new role `demo_wsnap` + root playbook `writable_snapshot_demo.yml`.** Isolated from `site.yml`,
  two layers, PPDM automation behind a flag.
- 2: extend `csm_k3s`/`csm_ocp`/`demo_wiki`. Rejected: puts the writable StorageClass in every `site.yml` run; mixes deploy and demo.
- 3: docs and manifests only. Rejected: not repeatable, against the repo's approach.

## Design

Play: `hosts: mgmt` (like `dr_test.yml`, `cleanup.yml`), primary site only. Role variables live in role defaults;
per-track data only in `platform.yml`.

Source StorageClass A: `dr_protect_sc` is a `set_fact` inside the `dr_protect` role (it picks the StorageClass with
`isReplicationEnabled`), so it is undefined in a standalone play. The role discovers A the same way
(`kubernetes.core.k8s_info` on StorageClass), overridable with `-e demo_wsnap_source_sc=...`; the play fails with a clear
message if none is found.

StorageClass B parameters are **read from A** (`IsiPath`, `AccessZone`, `AzServiceIP`, `ClusterName`), never from
shared vars: `AccessZone: System` is hard-coded in `roles/csm_ocp/templates/storageclass.yaml.j2`, not a variable.
B adds only `csi-isilon.dellemc.com/writable-from-snapshot: "true"`, `reclaimPolicy: Delete`, no replication or
retention parameters. Whether B needs `RootClientEnabled: "true"` (PPDM guide recommends a rootclient class for NFS
temporary PVCs) is decided on the lab; default copies A.

Data-mover namespace: `ppdm-custom-namespace-storage-class-mapping` gets one entry for A **and** one for B
(PPDM 20.3 guide: when the temporary-PVC mapping is used, the mapped class name must be in that ConfigMap). Variable
`demo_wsnap_datamover_ns`; its pods must reach the AccessZone of A and B. This makes the cross-namespace risk below likely,
not hypothetical.

Variables: `demo_wsnap_sc_name` (B), `demo_wsnap_source_sc` (A, discovered), `demo_wsnap_namespace`, `demo_wsnap_pvc` (`pgdata`), `demo_wsnap_ppdm_automate` (false),
`demo_wsnap_ppdm_host`, `demo_wsnap_ppdm_user`, `demo_wsnap_ppdm_password` (vault/env), `demo_wsnap_poll_seconds`.

Task files: `preflight.yml` (versions, flags), `storage.yml` (B + VolumeSnapshotClass), `csm_check.yml`
(write marker file -> snapshot -> change marker -> clone PVC -> read old marker from a dedicated pod -> array assert -> cleanup), `ppdm_config.yml` (ConfigMaps + controller restart),
`ppdm_api.yml` (login, register, policy, start, follow activity; adapted from bottkars/ansible_ppdm 130.x/31.1 with
provenance comment: repo URL + commit SHA), `evidence.yml` (poll + timeline + verdict), `cleanup.yml` (needs `-e confirm_writable_cleanup=yes`).

VolumeSnapshotClass (call, on the user's word "PPDM can have our snapshot class"): reuse the class the repo already
applies (K3s: `csm_k3s/tasks/prereqs.yml` applies `04_volumesnapshotclass.yaml` from the dellctl manifest); do not create
a second one. `ppdm_config.yml` renders `ppdm-snapshot-class-mapping` as `<A>: <that class>` so PPDM snapshots with
it even if several classes exist for `csi-isilon.dellemc.com`. The class name is read from the cluster
(`kubectl get volumesnapshotclass`), not hard-coded. On OpenShift the repo applies none: `storage.yml` creates one
only if the cluster has none. `storage.yml` therefore creates StorageClass B, and a VolumeSnapshotClass only when absent. The csm_k3s task applies
it per site; the demo uses the primary site's only. Selection is deterministic: list classes with driver
`csi-isilon.dellemc.com`; exactly one -> use it; none -> create; several -> require `-e demo_wsnap_snapclass=...`.
Builder note: pick one PPDM OpenAPI spec version on purpose (21201 = v2 login/clusters/rules/activities, 21202 = v3
policies/activities; 21200 has none of them); check whether following the job needs v3 activities.
Soften one claim: `resizer.enabled` is required only for volume expansion (CSM guide note), not for the feature.

Array assert: `ps_access: api` -> `GET /platform/14/snapshot/writable`; `ssh` -> `isi snapshot writable list`.

## Known risks to watch on the lab (record, do not assume)

1. PPDM creates the temporary PVC in a different namespace than the snapshot -> cross-namespace limit.
2. PPDM deletes the VolumeSnapshot before the temporary PVC -> `SnapshotHasDependents`. The PPDM 20.3 guide
   ("Warning displays when temporary persistent volumes not deleted by CSI driver before timeout") confirms PPDM
   creates a temporary PV/PVC per snapshot and waits only 60 s for its removal; a slow CSI delete ends the backup
   with a warning `PV <name> of snapshot <name> might not be deleted. Context deadline exceeded.` Watch for it, and
   check OneFS for leftover writable directories after the job.
3. Backup path is FCD/nonsnapshot/ROX, so the mapping does not apply.
4. K3s/K8s 1.34 not supported by PPDM 20.3; on K3s PPDM may snapshot with its own class and ignore the mapping.
5. The temporary PVC on B needs NFS root access (rootClient) or ROX mode compatible with B; the PPDM guide recommends a
   rootclient class for NFS temporary PVCs.
6. The ConfigMaps live in `powerprotect` on the cluster PPDM protects. Rolling out `powerprotect-controller` after a
   change affects every workload that PPDM protects there. Do it on a lab only, say so in the user guide.
8. CSI snapshots may not be available for replicated volumes (the lab guide v6 says "not for replicated volumes"). `pgdata` is on the
   replicated class A, so PPDM may be unable to snapshot it; the protected PVC is therefore a variable
   (`demo_wsnap_protected_ns`, `demo_wsnap_protected_pvc`, with `demo_wsnap_source_sc` for its class). Ask Dell which PVs they mean.
7. False OBSERVED: a writable snapshot left by the CSM-layer test would satisfy the PPDM poll. The CSM-layer cleanup is
   verified (list empty for our source snapshots) before the PPDM job; evidence is filtered by source snapshot name and
   by creation time after the job start.

Note: risk 2 (ordering) is our inference. The guide only describes a slow delete producing a warning.

## PPDM API automation: off by default (decided with the user)

`demo_wsnap_ppdm_automate` defaults to `false`. Why: the CSM layer must run alone, without a PPDM host or credentials, and
prove the PowerScale half first; and the REST calls (register cluster, create policy) change PPDM, so they run only
when someone turns the flag on. With the flag off, the ConfigMaps are still rendered and the PPDM steps are done in
the PPDM UI (checklist in the user guide). When on, it needs `demo_wsnap_ppdm_host`, user and password (env or vault,
never in files).

Calls made by me, not by the user (check them):
- With the flag on, requests that change PPDM configuration also need `-e confirm_ppdm_changes=yes`, like the DR
  playbooks need their `confirm_...=yes`. Starting a backup and reading activities need no confirmation.
- Cluster registration is only attempted if the cluster is not already registered (idempotency to be checked in the
  20.3 OpenAPI by the builder).
- Marker test for the CSM layer runs in a dedicated pod mounted on the clone, not in the Wiki.js/Postgres pod.
- Both tracks are tried; K3s is not named in the PPDM 20.3 guide.
- The walkthrough lives in its own file `docs/WRITABLE_SNAPSHOT_DEMO.md` and the evidence timeline is generated as a
  Mermaid file by the playbook (the user asked for states and Mermaid, "or better"; generated-from-the-run is my reading of "better").

## Left to the builder

Exact PPDM request bodies (from the 20.3 OpenAPI), ConfigMap key spelling check against the guide, timeline file format,
Molecule: none possible (needs arrays) — template golden test like `tests/test_dellctl_template.yml`.

## Implementation deviations (rulings made while building, 2026-11-15)

Two of these change what was approved and need the user's eye: R5 and R9.

- **R9 (changes the approved scope):** the REST automation does not register the cluster or create the policy. It logs in, asserts a
  Kubernetes cluster is registered, finds an EXISTING policy by name (`demo_wsnap_ppdm_policy`), starts it with
  `POST /api/v3/protections` (the older `/protection-policies/{id}/protections` used by bottkars is absent from the 20.3 OpenAPI) and follows the
  activity. Reason: the nested v3 policy body and the Kubernetes registration credential cannot be verified offline. Registration and policy stay in the PPDM UI.
- **R5 (changes the approved scope):** the CSM-layer test uses a scratch PVC `demo-ws-src` on a new plain StorageClass C, not `pgdata` on A:
  a marker file inside the Postgres data directory would corrupt it, and a PVC on the replicated class would create RG/SyncIQ side effects.
  The PPDM layer still protects the real `pgdata`.
- R1: evidence is keyed on the array `Src Path` of the protected volume directory (derived from the PV `volumeHandle`, format to confirm on the lab)
  plus a baseline diff, not on snapshot names (backend snapshot name differs from the VolumeSnapshot name) nor on creation time.
- R2: OneFS API fields (OAS 9.13): `created` (epoch), `dst_path`, `src_path`, `src_snap`, `state`; the CLI table has no Created column, so ssh mode asks for `--format json` (to confirm on the lab).
- R3: parsing is in `tasks/array_parse.yml`, pure and tested; `array_list.yml` only fetches.
- R4: preflight checks the `snapshotter` and `resizer` containers of the `isilon-controller` pod (same on both tracks), `kubectl version -o json`,
  strips suffixes such as `+k3s1`, and accepts digest-pinned images (tag extraction in `tasks/image_tag.yml`, tested).
- R7: pod image is the existing `demo_image`.
- R8: variables are `demo_wsnap_*` (ansible-lint `var-naming` requires the role name as prefix); the spec was renamed accordingly.
- R10: regex back-references are `\1` in `{{ }}` values and `\\1` in `that:` expressions (verified).
- R11: the cleanup phase does not run discovery, so a missing replicated class cannot block cleanup.
- The evidence loop guard lives inside `evidence_tick.yml`: a `when` on an `include_tasks` loop is evaluated for all items before any runs.
- Evidence also reports new writable snapshots on unwatched paths as a clue when the verdict is NOT OBSERVED.
- SnapshotIQ licensing is left to `ps_prep` (already asserted by `preflight.yml`).
- The protected PV is configurable (`demo_wsnap_protected_ns` / `_pvc`, default the first Wiki.js namespace and `pgdata`) because of risk 8.
- The watched OneFS path is read from `spec.csi.volumeAttributes.Path` of the PV (replicated volumes sit under a replication-group folder);
  the `IsiPath/<handle>` guess is only a fallback.
- The evidence phase refuses REST mode unless `--tags ppdm` ran in the same run, and takes the cluster baseline before the loop so existing objects are not events.
- Deletion is awaited at PV level (`kubectl wait --for=delete pv/...`) before the array is re-listed.
- Deferred minors: choose the backup objective by type instead of `objectives[0]`; fail when `demo_wsnap_snapclass` names a class of another driver;
  assert in `ppdm_config.yml` that the VolumeSnapshotClass exists when `--tags ppdm` runs alone; table parsing assumes three columns (only used if `--format json` is not honoured).
- Preflight probes a CSI snapshot of the protected PVC (create, wait 90 s, delete) and only warns, to test the lab guide v6 rule "no CSI snapshots of replicated volumes"; it snapshots the PVC itself so no replicated volume is created. Flag `demo_wsnap_probe_snapshot` (default true).
- **R5 revised (user: "il faut la réplication pour la démo"):** the scratch PVC and the plain class C are gone. The CSM layer snapshots the protected
  REPLICATED PV itself (`pgdata`), clones it on B in the same namespace, lists and writes the clone from a pod, checks the array, and cleans up.
  The source is never written; no new replicated volume or RG is created. B now sets `RootClientEnabled: "true"` (the PPDM guide recommends a rootClient
  class for NFS temporary PVCs; the data mover reads the clone as root). If CSI snapshots of replicated volumes do not work (lab guide v6 rule), the CSM layer
  fails at the snapshot with that message, which is the finding to take to Dell.
- **R9 lifted (user: register and create the policy with Ansible; lab values as placeholders):** `--tags register` (never by default; needs
  `demo_wsnap_ppdm_automate=true` and `confirm_ppdm_changes=yes`) applies the PPDM RBAC archive, creates the service-account token Secret, then through
  the REST API creates the Kubernetes credential, accepts the API server certificate, adds the asset source, starts a discovery, waits for the namespace asset,
  finds the Data Domain storage system, creates a v3 `KUBERNETES` policy (on demand, full, 1 day) and assigns the namespace. Each step reuses what exists.
  Lab values are placeholders in `inventory/<track>/group_vars/all/platform.yml`. All 17+ operations are checked against the 20.3 OpenAPI by
  `tests/test_writable_snap_endpoints.yml`; the bodies themselves follow the OpenAPI schema and bottkars and are unverified against a live PPDM
  (the namespace asset type "KUBERNETES" in the discovery wait is to confirm).
