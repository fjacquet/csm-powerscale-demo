# PowerScale writable snapshots behind PPDM backups — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A repeatable Ansible experiment that observes (or fails to observe) a OneFS writable snapshot during a PPDM 20.3 backup of a PV, on the K3s and OpenShift tracks.

**Architecture:** New role `roles/demo_wsnap` + root playbook `writable_snapshot_demo.yml` (never reached by `site.yml`). Two layers: CSM layer (StorageClass B writable, snapshot -> clone -> array check, no PPDM needed) and PPDM layer (ConfigMaps in `powerprotect`, optional REST automation). `evidence.yml` polls cluster and array during the PPDM job and ends with OBSERVED / NOT OBSERVED plus a generated Mermaid timeline.

**Tech Stack:** Ansible (kubernetes.core `k8s_info`, `kubectl apply` of rendered manifests as in `demo_wiki`, `ansible.builtin.uri`, `ansible.builtin.raw` for `isi`), Jinja2, Mermaid. No git repo here: no commit steps; each task ends with its validation command as the checkpoint.

**Spec:** `docs/superpowers/specs/2026-11-15-writable-snapshot-ppdm-design.md`

**Plan decisions beyond the spec (check them):**
- The CSM-layer marker test uses a dedicated scratch PVC `demo-ws-src` on StorageClass A in its own namespace `demo-ws` (var `demo_wsnap_namespace`), not `pgdata`: writing a marker file into the Postgres data directory would corrupt it. The PPDM layer protects the real Wiki.js namespace (`demo_wiki_namespaces`, first entry). A on the replication class for a PVC outside any RG is untested: the lab decides.
- Array access reuses `ps_access`: `api` -> `GET https://<endpoint>:8080/platform/14/snapshot/writable` with `ansible.builtin.uri` (dellemc.powerscale has no writable-snapshot module); `ssh` -> `isi snapshot writable list` via `ansible.builtin.raw` delegated to the array host, as `roles/ps_prep/tasks/array_ssh.yml` does.

## Global Constraints

- Playbook at the repo root; `hosts: mgmt`; primary site only (`sites.primary`).
- Role variables prefixed `demo_wsnap_`; defaults in `roles/demo_wsnap/defaults/main.yml`; per-track data only in `inventory/<t>/group_vars/all/platform.yml`; never `platform`/`sites`/`arrays` in the shared file.
- Jinja only at the end of task names. ansible-lint profile `production` clean; yamllint clean.
- `site.yml` must never import the new playbook; guards: cleanup needs `-e confirm_writable_cleanup=yes`; PPDM configuration writes need `-e confirm_ppdm_changes=yes`.
- Versions: OneFS >= 9.13, Kubernetes >= 1.34, csi-isilon driver >= 2.18.0, external-snapshotter >= v8.4.0, PPDM 20.3.
- StorageClass B: `csi-isilon.dellemc.com/writable-from-snapshot: "true"`, `reclaimPolicy: Delete`, `volumeBindingMode: Immediate`, `IsiPath`/`AccessZone`/`AzServiceIP`/`ClusterName` copied from A, no replication or retention parameters.
- ConfigMaps in namespace `powerprotect`: `ppdm-snapshot-class-mapping` (A -> VolumeSnapshotClass), `ppdm-snapshot-storage-class-mapping` (A -> B), `ppdm-custom-namespace-storage-class-mapping` (A and B -> `demo_wsnap_datamover_ns`).
- `demo_wsnap_ppdm_automate` defaults to `false`; secrets from env or vault, `no_log` on every task that carries them.
- No bottkars repo dependency; copied logic carries a provenance comment (https://github.com/bottkars/ansible_ppdm + commit SHA, fetched with `gh api repos/bottkars/ansible_ppdm/commits/main --jq .sha`).
- Diagrams are Mermaid; no ASCII-art tables.
- Run `superpowers:verification-before-completion` before saying done.

## Review Focus

- No StorageClass with `isReplicationEnabled` exists, or several do: fail with a message naming `-e demo_wsnap_source_sc=...`; never pick silently.
- A has `AccessZone` other than `System`: B must carry that zone (test with `AccessZone: zone-x`).
- Zero or several `csi-isilon` VolumeSnapshotClasses: create one / require `-e demo_wsnap_snapclass=...`.
- A writable snapshot from an earlier run is still on the array: evidence must not count it (filter by source-snapshot name and creation time after job start).
- Array returns an empty list (`Total: 0`, empty JSON `writable`): parsed as zero entries, not an error.
- `demo_wsnap_ppdm_automate: true` with empty host or password: fail before any request; password never printed.

## File Structure

- Create `roles/demo_wsnap/{defaults,meta,tasks,templates}` (scaffold with the `ansible-new-role` skill).
- `tasks/`: `main.yml` (dispatch by tag), `preflight.yml`, `discover.yml` (A, snapshot class), `storage.yml`, `array_list.yml`, `csm_check.yml`, `ppdm_config.yml`, `ppdm_api.yml`, `evidence.yml`, `cleanup.yml`.
- `templates/`: `storageclass_b.yaml.j2`, `configmaps.yaml.j2`, `csm_check.yaml.j2`, `timeline.mmd.j2`.
- Create `writable_snapshot_demo.yml` (root).
- Create `tests/test_writable_snap_templates.yml`, `tests/test_writable_snap_parse.yml`, `tests/test_writable_snap_endpoints.yml`.
- Create `docs/WRITABLE_SNAPSHOT_DEMO.md`; modify `docs/USER_GUIDE.md`, `README.md`, add `roles/demo_wsnap/README.md`.

---

### Task 1: Role skeleton, argument specs, playbook shell

**Files:** Create the role tree and `writable_snapshot_demo.yml`; Modify `.ansible-lint` only if the new role needs an exclusion (it should not).

**Interfaces:** Produces role entry `demo_wsnap` with `tasks_from` = `preflight|storage|csm_check|ppdm_config|ppdm_api|evidence|cleanup`; variables `demo_wsnap_sc_name`, `demo_wsnap_source_sc`, `demo_wsnap_namespace`, `demo_wsnap_snapclass`, `demo_wsnap_datamover_ns`, `demo_wsnap_ppdm_automate`, `demo_wsnap_ppdm_host`, `demo_wsnap_ppdm_user`, `demo_wsnap_ppdm_password`, `demo_wsnap_poll_seconds`, `demo_wsnap_pod_image`, `demo_wsnap_pvc_size`, `demo_wsnap_evidence_timeout`.

- [ ] **Step 1:** Invoke the `ansible-new-role` skill to scaffold `roles/demo_wsnap`; keep only the directories listed above.
- [ ] **Step 2:** Write `meta/argument_specs.yml` for every variable above (types, defaults: `demo_wsnap_sc_name: powerscale-writable-snapshot`, `demo_wsnap_namespace: demo-ws`, `demo_wsnap_datamover_ns: powerprotect`, `demo_wsnap_ppdm_automate: false`, `demo_wsnap_poll_seconds: 5`, `demo_wsnap_pod_image: busybox:1.36`, `demo_wsnap_pvc_size: 1Gi`, `demo_wsnap_evidence_timeout: 1800`; password has no default, `no_log` noted in description).
- [ ] **Step 3:** Write `writable_snapshot_demo.yml`: one play, `hosts: mgmt`, one `import_role` per phase with tags `preflight`, `csm`, `ppdm`, `evidence`, and `cleanup` + `never`; each phase file is an empty `- name: ...ansible.builtin.debug` placeholder.
- [ ] **Step 4:** Validate with the `ansible-docs` skill that `import_role` + `tasks_from` + tags usage is correct, then run: `uv run ansible-playbook -i inventory/k3s --syntax-check writable_snapshot_demo.yml` and the same with `inventory/openshift`. Expected: `playbook: writable_snapshot_demo.yml`, exit 0.
- [ ] **Step 5:** Run `uv run ansible-lint && uv run yamllint .` Expected: no violations.

### Task 2: Templates and offline golden test (StorageClass B, ConfigMaps)

**Files:** Create `templates/storageclass_b.yaml.j2`, `templates/configmaps.yaml.j2`, `tests/test_writable_snap_templates.yml`.

**Interfaces:** Template input: `demo_wsnap_source` (dict: `name`, `parameters`), `demo_wsnap_sc_name`, `demo_wsnap_snapclass`, `demo_wsnap_datamover_ns`. Produces a StorageClass and one multi-document ConfigMaps file.

- [ ] **Step 1:** Write the failing test (same shape as `tests/test_wiki_templates.yml`, vars_files for both tracks, `demo_wsnap_source` fixture with `parameters` `IsiPath: /ifs/data/csi`, `AccessZone: zone-x`, `AzServiceIP: 192.168.1.20`, `ClusterName: ps1` plus replication keys). Assertions: `test_b_copies_zone` (B has `AccessZone: zone-x` and the fixture `IsiPath`), `test_b_writable_flag`, `test_b_delete_reclaim`, `test_b_no_replication_keys` (no key containing `replication.storage.dell.com`), `test_cm_class_mapping` (data `{A}: {snapclass}`), `test_cm_sc_mapping` (`{A}: {B}`), `test_cm_namespace_mapping_has_both` (entries for A and B with `demo_wsnap_datamover_ns`), all in namespace `powerprotect`.
- [ ] **Step 2:** `uv run ansible-playbook tests/test_writable_snap_templates.yml` -> Expected: FAIL (templates missing).
- [ ] **Step 3:** Implement the two templates; the parameters block iterates `demo_wsnap_source.parameters` keeping only the four copied keys.
- [ ] **Step 4:** Re-run -> Expected: PASS on both tracks. Then `uv run ansible-lint`.

### Task 3: Array listing and evidence parsing (offline-testable)

**Files:** Create `tasks/array_list.yml`, `tests/test_writable_snap_parse.yml`; fixture text/JSON inline in the test.

**Interfaces:** Produces fact `demo_wsnap_writable` = list of `{path, src_path, src_snapshot, created}` (empty list when none). Consumes `ps_access`, `ps_conn`, `arrays`, `sites.primary.local_array`. Filter helper in the role as a `vars` expression: entries whose `src_snapshot` is in `demo_wsnap_watch_snapshots` and `created >= demo_wsnap_job_start`.

- [ ] **Step 1:** Write failing tests: `test_parse_ssh_table` (the `isi snapshot writable list` layout from the CLI guide with three rows -> 3 entries, `Total: 0` -> empty list), `test_parse_api_json` (`{"writable": [...]}` per the OneFS 9.13 OAS schema of `GET /platform/14/snapshot/writable`; read the response schema from `docs/openapi/91300_QUICKSTEP_OAS3-9.13.0.json` to pin field names, including the creation-time field), `test_filter_excludes_stale` (an entry with older `created` or other snapshot name is dropped).
- [ ] **Step 2:** Run -> Expected: FAIL.
- [ ] **Step 3:** Implement `array_list.yml`: `ssh` branch `ansible.builtin.raw: isi snapshot writable list --format json` if `--format json` is supported by the CLI guide, else parse the table; `api` branch `ansible.builtin.uri` GET with basic auth from `ps_conn`, `no_log: true`, `validate_certs` from `skip_cert_validation`. Use `ansible-docs` to confirm `uri` parameters. Parsing/filtering stays in Jinja expressions inside the file so the test imports the same code.
- [ ] **Step 4:** Run -> Expected: PASS. `uv run ansible-lint`.

### Task 4: Preflight and discovery

**Files:** Create `tasks/preflight.yml`, `tasks/discover.yml`.

**Interfaces:** Produces `demo_wsnap_source` (A as `k8s_info` StorageClass resource) and `demo_wsnap_snapclass` (selected class name). Consumes `sites.primary.kubeconfig`.

- [ ] **Step 1:** `discover.yml`: `kubernetes.core.k8s_info` StorageClass; A = `demo_wsnap_source_sc` if set, else the single class with `parameters['replication.storage.dell.com/isReplicationEnabled']`; `ansible.builtin.fail` with the `-e demo_wsnap_source_sc=...` message on 0 or >1. List VolumeSnapshotClass with driver `csi-isilon.dellemc.com`: one -> use; none -> mark `demo_wsnap_create_snapclass: true`; several -> require `demo_wsnap_snapclass`.
- [ ] **Step 2:** `preflight.yml` assertions with clear `fail_msg`: K8s server version >= 1.34 (`k8s_info` on the `version` API or `kubectl version -o json`); csi-isilon controller image tag >= 2.18.0 in namespace `csi_ns`; helm values `snapshot.enabled` true; snapshot-controller deployment present with image >= v8.4.0; OneFS version >= 9.13 (`ssh`: `isi version`; `api`: `GET /platform/3/cluster/version`); SnapshotIQ licensed (`isi license list` / licensing API). On OpenShift (`platform == 'openshift'`) also read the OCP version and print it (no gate).
- [ ] **Step 3:** Use `ansible-docs` to validate `k8s_info` parameters; run syntax-check on both inventories, `uv run ansible-lint`.

### Task 5: Storage apply and CSM-layer check

**Files:** Create `tasks/storage.yml`, `tasks/csm_check.yml`, `templates/csm_check.yaml.j2`.

**Interfaces:** `storage.yml` applies B (and a VolumeSnapshotClass when `demo_wsnap_create_snapclass`) with `kubectl --kubeconfig ... apply -f` of files rendered under `{{ dr_dir }}/writable/`. `csm_check.yml` produces the pass/fail of the CSM layer and registers `demo_wsnap_watch_snapshots` = `[demo-ws-snap]`.

- [ ] **Step 1:** `storage.yml` renders and applies; StorageClass is skipped (not re-applied) when it already exists, because parameters are immutable.
- [ ] **Step 2:** `csm_check.yaml.j2` renders: namespace `demo_wsnap_namespace`, scratch PVC `demo-ws-src` on A (size `demo_wsnap_pvc_size`), a writer pod that writes `/data/marker` = `v1`, VolumeSnapshot `demo-ws-snap`, then (applied in a later step) pod mutator writing `v2`, clone PVC `demo-ws-clone` on B with `dataSource` the snapshot, reader pod `cat /data/marker`.
- [ ] **Step 3:** `csm_check.yml` sequence with waits (`kubectl wait`): write v1 -> snapshot readyToUse -> write v2 -> clone Bound -> reader prints `v1` (assert) -> `array_list.yml` shows a writable snapshot whose `src_snapshot` matches the backing snapshot -> assert present -> delete reader, clone PVC, snapshot, source PVC (this order) -> `array_list.yml` again -> assert none left for `demo-ws-snap`. Failure leaves resources for inspection and prints the state diagram step name from `docs/WRITABLE_SNAPSHOT_DEMO.md`.
- [ ] **Step 4:** Syntax-check both inventories, `uv run ansible-lint`. (Cannot run without clusters: state this in the task report.)

### Task 6: PPDM ConfigMaps

**Files:** Create `tasks/ppdm_config.yml`.

- [ ] **Step 1:** Render `configmaps.yaml.j2` to `{{ dr_dir }}/writable/ppdm-configmaps.yaml`, show the diff-able file path, `kubectl apply` it only with `confirm_ppdm_changes == 'yes'`; without confirmation, print the file path and the exact `kubectl apply` command and stop the phase with a clear message.
- [ ] **Step 2:** After apply, `kubectl rollout restart deployment powerprotect-controller -n powerprotect` and `rollout status`; print the warning that other protected workloads are affected.
- [ ] **Step 3:** Syntax-check, `uv run ansible-lint`.

### Task 7: PPDM REST automation (flag on)

**Files:** Create `tasks/ppdm_api.yml`, `tests/test_writable_snap_endpoints.yml`.

**Interfaces:** Consumes `demo_wsnap_ppdm_host|user|password`. Produces `demo_wsnap_job_start` (UTC ISO string) and `demo_wsnap_activity_id`.

- [ ] **Step 1:** Write the failing test `test_endpoints_exist_in_openapi`: for each path/method pair the role uses, assert it exists in `docs/openapi/21201-20.3.0.json` or `21202-20.3.0.json` — `POST /api/v2/login`, `GET /api/v2/kubernetes-clusters`, `POST /api/v2/inventory-sources`, `POST /api/v3/protection-policies`, `POST /api/v2/protection-policies/{id}/asset-assignments`, `GET /api/v3/activities/{id}` (resolve the "start a policy run" operation by searching both specs, the way bottkars `130.4_playbook_start_k8s_policy.yaml` calls it, and add it to the list; if it is not in the 20.3 spec, record that in the test as a failing finding for the user).
- [ ] **Step 2:** Run -> Expected: FAIL for the start endpoint until resolved.
- [ ] **Step 3:** Implement `ppdm_api.yml`: guard first (`fail` when host or password empty); `uri` login on `https://<demo_wsnap_ppdm_host>:8443/api/v2/login` (port 8443 to confirm against the `servers` entry of `21201-20.3.0.json`; the test reads it and fails on mismatch); token kept in a registered var with `no_log`; register cluster only when `kubernetes-clusters` does not list it, and only with `confirm_ppdm_changes == 'yes'`; create the policy on the Wiki.js namespace and assign assets (same guard); start the run, set `demo_wsnap_job_start`, poll the activity until a terminal state or timeout. Header comment: bottkars provenance with the SHA. Use `ansible-docs` for `uri` (`status_code`, `body_format`, `validate_certs`), and `ansible-good-practices` for naming.
- [ ] **Step 4:** Re-run the test -> Expected: PASS (or the explicit finding). Syntax-check both inventories, lint.

### Task 8: Evidence, verdict, Mermaid timeline

**Files:** Create `tasks/evidence.yml`, `templates/timeline.mmd.j2`; extend `tests/test_writable_snap_parse.yml`.

**Interfaces:** Consumes `demo_wsnap_job_start`, `demo_wsnap_watch_snapshots`, `demo_wsnap_poll_seconds`. Produces `demo_wsnap_verdict` (`OBSERVED` | `NOT OBSERVED`) and `{{ dr_dir }}/writable/timeline.mmd`.

- [ ] **Step 1:** Failing test `test_timeline_renders`: from a fixed event list (`{t, source: cluster|array, text}`) the template renders a `sequenceDiagram` with participants `Cluster` and `Array` and one message per event, in time order; `test_verdict` : empty filtered list -> `NOT OBSERVED`, one entry -> `OBSERVED`.
- [ ] **Step 2:** Run -> Expected: FAIL. Implement the template and verdict expression.
- [ ] **Step 3:** `evidence.yml`: first assert the CSM-layer leftovers are gone (`array_list.yml`, none for `demo-ws-snap`); loop every `demo_wsnap_poll_seconds` until the PPDM activity ends (or `demo_wsnap_evidence_timeout`, default 1800 s): record `kubectl get volumesnapshot,pvc -A -o name` deltas and filtered writable list; write the timeline; print the verdict and the path. Warn when the activity result contains `might not be deleted`.
- [ ] **Step 4:** Tests PASS; lint; syntax-check both inventories.

### Task 9: Cleanup

**Files:** Create `tasks/cleanup.yml`.

- [ ] **Step 1:** Assert `confirm_writable_cleanup == 'yes'`; delete in the order clone PVC, VolumeSnapshots, source PVC, namespace `demo_wsnap_namespace`; StorageClass B only with `-e demo_wsnap_remove_sc=yes`; verify through `array_list.yml` that no writable entry remains for the demo snapshots.
- [ ] **Step 2:** Syntax-check, lint; confirm `grep -n writable_snapshot_demo site.yml` returns nothing.

### Task 10: Documentation with Mermaid

**Files:** Create `docs/WRITABLE_SNAPSHOT_DEMO.md`, `roles/demo_wsnap/README.md`; Modify `docs/USER_GUIDE.md`, `README.md`.

- [ ] **Step 1:** `docs/WRITABLE_SNAPSHOT_DEMO.md`: `stateDiagram-v2` (Preflight -> CSM check -> PPDM configured -> Backup running -> Evidence collected -> OBSERVED / NOT OBSERVED, a failure state per step with the command that identifies it); `sequenceDiagram` of PPDM controller, snapshotter, csi-isilon, OneFS API, cProxy, with a note marking inferred steps; `flowchart` for the NOT OBSERVED decision (namespace, cleanup order, backup path, platform support, rootClient/ROX); per step: command, what the audience sees, expected output, what to say, as Markdown lists. Run commands: `ansible-playbook writable_snapshot_demo.yml --tags preflight,csm`, then `-e confirm_ppdm_changes=yes --tags ppdm,evidence` (with `-e demo_wsnap_ppdm_automate=true` for API mode), cleanup with `--tags cleanup -e confirm_writable_cleanup=yes`.
- [ ] **Step 2:** Add the user-guide part, README links, role README (variables table as a Markdown list).
- [ ] **Step 3:** Validate diagrams with `npx -y @mermaid-js/mermaid-cli@11 -i <file>` extraction if network allows, else report "not validated"; do not claim validation otherwise.

### Task 11: Final verification and reviews

- [ ] **Step 1:** Invoke `ansible-good-practices` and `ansible-zen` skills on the role; fix findings that trace to this change only.
- [ ] **Step 2:** Run all: `uv run ansible-lint && uv run yamllint . && uv run ansible-playbook tests/test_dellctl_template.yml && uv run ansible-playbook tests/test_writable_snap_templates.yml && uv run ansible-playbook tests/test_writable_snap_parse.yml && uv run ansible-playbook tests/test_writable_snap_endpoints.yml`, then syntax-check `writable_snapshot_demo.yml` and `site.yml` on both inventories and `uv run ansible-inventory -i inventory/k3s --host mgmt_vm --playbook-dir .`. Expected: all exit 0.
- [ ] **Step 3:** Invoke `superpowers:verification-before-completion`. Report plainly: nothing ran against real clusters, arrays or PPDM; list what is unvalidated (lab behavior, PPDM start endpoint if unresolved, Mermaid rendering).
- [ ] **Step 4:** Update the `Status` section of `CLAUDE.md` only if the user asks (it is the user's file).
