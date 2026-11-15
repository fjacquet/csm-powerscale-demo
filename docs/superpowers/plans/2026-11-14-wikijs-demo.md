# Wiki.js demo workload Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Run Wiki.js + PostgreSQL on `app-a` (both tracks), deployed with `kubectl apply`, reachable by URL on each site, with an automated marker-row consistency check across failover.

**Architecture:** New role `demo_wiki` renders Jinja manifests to `{{ dr_dir }}/wiki/<site>/` and applies them with `kubectl`. `dr_protect/protect_ns.yml` calls it instead of creating the busybox writer for namespaces in `demo_wiki_namespaces`. `dr_ops` dispatches capture/check to wiki variants for those namespaces. MetalLB (K3s) and Routes (OpenShift) give each site a URL.

**Tech Stack:** Ansible (ansible-core, `ansible.builtin.command`/`template`), kubectl, MetalLB native manifest (K3s), sclorg PostgreSQL image, `ghcr.io/requarks/wiki:2`.

**Spec:** `docs/superpowers/specs/2026-11-14-wikijs-demo-design.md` (read it, including "Amendment" and "Hard rule").

**No git repo:** this directory is not under git; "checkpoint" steps below run the verification commands instead of committing.

## Global Constraints

- Playbooks stay at repo root. Per-track data only in `inventory/<t>/group_vars/all/platform.yml`; never put `platform`/`sites`/`arrays` in `group_vars/all/main.yml`.
- Role variables carry the role prefix (`demo_wiki_`); task names have Jinja only at the end.
- `site.yml` never reaches a DR playbook; no `confirm_...=yes` guard is changed.
- `pgdata` PVC is applied on the primary ONLY. Only Secret, Services, Route/LoadBalancer go to both sites. Postgres is a Deployment, `strategy: Recreate`, PVC by `claimName: pgdata`, no `volumeClaimTemplates`.
- Deployment is `kubectl --kubeconfig <site kubeconfig> apply -f <file>`; `changed_when` from `created`/`configured` in stdout. One Postgres image on both tracks (sclorg/Red Hat). Wiki.js image `ghcr.io/requarks/wiki:2`, `emptyDir` on its `dataPath`. `anyuid` SCC only as documented lab fallback.
- DB password is ONE vault variable shared by both sites; never generated per site.
- Secrets and credential files: `no_log: true`, mode 0600.
- Verification gates (repo CLAUDE.md): `uv run ansible-lint && uv run yamllint .`, `--syntax-check` for BOTH inventories, golden test still passes.
- Nothing runs on real clusters: say so in docs; unverified facts (image env vars, finalize endpoint, NFS UID) are marked `verify` in variables or comments.

## Review Focus

- `demo_wiki_namespaces: []` -> busybox everywhere, no MetalLB, no URL tasks, no errors. (Task 3, 4)
- A namespace in `demo_wiki_namespaces` that is not in `app_namespaces` -> clear assert failure. (Task 1)
- `-e '{"only_namespaces":["app-b"]}'` -> wiki capture/check never runs for `app-a`. (Task 5)
- Unplanned failover: primary is down, so `psql` capture fails -> warn like the busybox path, do not abort. (Task 5)
- Re-running `site.yml` -> `kubectl apply` reports `unchanged`, no changed tasks, wizard bootstrap not re-run once finalized. (Task 3)

---

### Task 1: Variables and role skeleton

**Files:**
- Modify: `group_vars/all/main.yml` (section "DR demo workload")
- Modify: `group_vars/all/vault.yml.example`
- Modify: `inventory/k3s/group_vars/all/platform.yml` (add `lb_range` per site)
- Create: `roles/demo_wiki/{meta/main.yml,meta/argument_specs.yml,README.md,defaults/main.yml,tasks/main.yml}` (mirror `roles/dr_ops`)
- Modify: `roles/dr_protect/tasks/main.yml` (assert only)

**Interfaces:**
- Produces (shared, `main.yml`): `demo_wiki_namespaces: [app-a]`, `demo_wiki_image`, `demo_wiki_db_image` (comment `verify` tag/registry), `demo_wiki_db_user`, `demo_wiki_db_name`, `demo_wiki_db_password: "{{ vault_wiki_db_password }}"`, `demo_wiki_admin_email`, `demo_wiki_admin_password: "{{ vault_wiki_admin_password }}"`, `demo_wiki_bootstrap: true`, `demo_wiki_metallb_manifest_url` (pinned version, `verify`).
- Produces (K3s platform file): `sites.<site>.lb_range` (CIDR or `a-b`), placeholders in the lab's `192.168.1.x` range.

- [ ] **Step 1:** Add the variables above and the two `vault_wiki_*` placeholders to `vault.yml.example`; add `lb_range` to both sites in the K3s `platform.yml`. Do NOT add anything to `inventory/openshift`.
- [ ] **Step 2:** Create the role skeleton; `argument_specs.yml` documents every `demo_wiki_*` variable; README follows the repo role README layout (see `roles/dr_ops/README.md`).
- [ ] **Step 3:** In `dr_protect/tasks/main.yml` add an `ansible.builtin.assert` (tag `always`): `demo_wiki_namespaces | difference(app_namespaces) | length == 0`, with a fail_msg naming the stray namespaces.
- [ ] **Step 4: Verify**
  Run: `uv run ansible-inventory -i inventory/k3s --host mgmt_vm --playbook-dir . | grep -E "demo_wiki|lb_range"` ; same for `inventory/openshift` (no `lb_range`).
  Run: `uv run ansible-lint && uv run yamllint .`
  Expected: variables resolve on both inventories; lint clean.

### Task 2: Manifest templates and the render test

**Files:**
- Create: `roles/demo_wiki/templates/pvc.yaml.j2` (primary only), `workloads.yaml.j2` (primary only), `shared.yaml.j2` (both sites)
- Create: `tests/test_wiki_templates.yml` (model on `tests/test_dellctl_template.yml`; `vars_files` main.yml + each track's platform.yml; renders with `lookup('ansible.builtin.template', ...)` and parses with `from_yaml_all`)

**Interfaces:**
- Consumes: Task 1 variables; `ns`, `site` (a `sites` entry), `dr_protect_sc`, `demo_pvc_size`, `platform`.
- Produces: templates rendering objects in namespace `ns`: `pvc.yaml.j2` -> PVC `pgdata` (RWX, `storageClassName: {{ dr_protect_sc }}`); `workloads.yaml.j2` -> Deployment `postgres` (Recreate, volume `claimName: pgdata`) and Deployment `wiki` (`DB_TYPE=postgres`, `DB_HOST=postgres`, `emptyDir` for `dataPath`); `shared.yaml.j2` -> Secret `wiki-db`, Services `postgres` and `wiki`, plus `Service type LoadBalancer` + `metallb.io` annotation (K3s) or `Route` (OpenShift), chosen by `platform`. MetalLB pool objects are NOT here (Task 4).

- [ ] **Step 1: Write the failing test** `tests/test_wiki_templates.yml`, one loop over `[k3s, openshift]` tracks and sites, asserting:
  - `shared.yaml.j2` rendered for both sites has no `PersistentVolumeClaim` and no `Deployment` (the hard rule);
  - `pvc.yaml.j2` renders exactly one object, name `pgdata`; `workloads.yaml.j2` has the `postgres` Deployment with `spec.strategy.type == 'Recreate'`, the volume `claimName == 'pgdata'`, and no `volumeClaimTemplates` anywhere;
  - K3s render contains a Service `type: LoadBalancer` and no `Route`; OpenShift render contains a `Route` and no LoadBalancer.
- [ ] **Step 2: Run** `uv run ansible-playbook tests/test_wiki_templates.yml` — Expected: FAIL (templates missing).
- [ ] **Step 3:** Implement the three templates. Image, env var names and data path for the sclorg image (`POSTGRESQL_USER/PASSWORD/DATABASE`, `/var/lib/pgsql/data`) come from the image README for the pinned tag: confirm before writing (`verify`). Postgres/Wiki read credentials from Secret `wiki-db` via `envFrom`.
- [ ] **Step 4: Run** the test again — Expected: PASS. Also run the existing golden test `uv run ansible-playbook tests/test_dellctl_template.yml` — Expected: PASS.

### Task 3: Deploy tasks and wiring into protect

**Files:**
- Create: `roles/demo_wiki/tasks/deploy.yml` (needs `ns`, `dr_protect_sc`), `roles/demo_wiki/tasks/apply.yml` (one rendered file + one site)
- Modify: `roles/dr_protect/tasks/protect_ns.yml` (busybox PVC+Deployment only `when: ns not in demo_wiki_namespaces`; add `include_role: demo_wiki tasks_from: deploy.yml` `when: ns in demo_wiki_namespaces`)

**Interfaces:**
- Consumes: Task 2 templates.
- Produces: `deploy.yml`: for each of `pvc`, `workloads` render to `{{ dr_dir }}/wiki/primary/<ns>-<name>.yaml` and apply on primary; render `shared` per site in `site_list` to `{{ dr_dir }}/wiki/<site.id>/` and apply on each; wait `kubectl -n <ns> rollout status deploy/postgres deploy/wiki --timeout=300s` on primary; bootstrap (below).
- Produces: `apply.yml(dr_wiki_file, dr_wiki_site)`: `kubectl --kubeconfig <site> apply -f <file>`, `register`, `changed_when: "'created' in stdout or 'configured' in stdout"`; file mode 0600 for rendered files holding the Secret, `no_log` on the Secret apply.

- [ ] **Step 1:** Implement `apply.yml` and `deploy.yml` as specified; namespace creation stays in `protect_ns.yml` (it runs first for both sites).
- [ ] **Step 2:** Bootstrap, `when: demo_wiki_bootstrap`: skip if the wiki already answers as finalized (e.g. `GET /` not the setup page); else POST the Wiki.js 2 `/finalize` request (adminEmail, adminPassword, adminPasswordConfirm, siteUrl set to the primary URL, telemetry false) via `kubectl exec deploy/wiki`. Confirm the endpoint and payload against Wiki.js 2 source/docs first (`verify`); if it fails, the task prints the URL and fails with the documented manual-wizard fallback message instead of aborting the whole run (`failed_when: false` + a `debug`).
- [ ] **Step 3: Wire protect_ns.yml** as described. Add the `when` guard so `demo_wiki_namespaces: []` yields today's behavior.
- [ ] **Step 4: Verify**
  Run: `uv run ansible-playbook -i inventory/k3s --syntax-check site.yml` and the same with `inventory/openshift`; `uv run ansible-lint`.
  Expected: clean. With `-e '{"demo_wiki_namespaces":[]}'` the syntax check still passes (guard).

### Task 4: MetalLB (K3s only)

**Files:**
- Create: `roles/demo_wiki/tasks/metallb.yml`, `roles/demo_wiki/templates/metallb-pool.yaml.j2`
- Modify: `roles/demo_wiki/tasks/main.yml` (include `metallb.yml` when `platform == 'k3s' and demo_wiki_namespaces | length > 0`), `site.yml` (add `role: demo_wiki` after the install track roles and before `dr_protect`, tag `demo_wiki`)

**Interfaces:**
- Consumes: `sites.<id>.lb_range`, `demo_wiki_metallb_manifest_url`.
- Produces: per site, `kubectl apply -f <metallb manifest URL>` once, `kubectl -n metallb-system wait --for=condition=Available deploy/controller --timeout=180s`, then apply a rendered `IPAddressPool` + `L2Advertisement` using that site's `lb_range`.

- [ ] **Step 1: Write the failing test:** extend `tests/test_wiki_templates.yml` with a render of `metallb-pool.yaml.j2` per K3s site; assert one `IPAddressPool` whose `addresses` equals `[lb_range]` and one `L2Advertisement`. Run: expect FAIL.
- [ ] **Step 2:** Implement the template and `metallb.yml`; run the test: expect PASS.
- [ ] **Step 3:** Add `role: demo_wiki` to `site.yml`; no tasks run on OpenShift or when the list is empty.
- [ ] **Step 4: Verify:** syntax-check `site.yml` for both inventories; lint.

### Task 5: Marker-row check in dr_ops

**Files:**
- Create: `roles/dr_ops/tasks/capture_wiki_ns.yml`, `roles/dr_ops/tasks/check_wiki_ns.yml`
- Modify: every caller of `capture_last_write.yml` and `check_data_ns.yml` under `roles/dr_ops/tasks/` and the root playbooks (grep for both names first; `failover_ns.yml`, `check_data.yml` are known callers)

**Interfaces:**
- Consumes: `ns`, `dr_site`, `dr_ops_check_site`, `dr_dir`, `demo_wiki_db_user`, `demo_wiki_db_name`.
- Produces: `capture_wiki_ns.yml`: runs `kubectl -n <ns> exec deploy/postgres -- psql -U <user> -d <db> -tA -c "CREATE TABLE IF NOT EXISTS dr_marker(ts text); INSERT INTO dr_marker VALUES (now()::text) RETURNING ts;"` on the `dr_site` cluster and stores the returned value in `{{ dr_dir }}/last-{{ ns }}.txt` (same file contract as the busybox path). Failure -> warn message, no abort (same as `capture_last_write.yml`).
- Produces: `check_wiki_ns.yml`: selects the row from `dr_marker` on `sites[dr_ops_check_site | default('secondary')]`, `until` success, `retries: 30`, `delay: 10`; asserts it equals the stored value; fail_msg names the namespace and "check the final sync / RPO".
- Dispatch rule at each caller: `include_tasks: "{{ 'capture_wiki_ns.yml' if ns in demo_wiki_namespaces else 'capture_last_write.yml' }}"` (likewise for check).

- [ ] **Step 1:** Grep callers (`grep -rn "capture_last_write\|check_data_ns" roles *.yml`); list them in the task notes.
- [ ] **Step 2:** Implement the two files with the contracts above; apply the dispatch at every caller found.
- [ ] **Step 3: Verify:** syntax-check every DR playbook (`failover_planned.yml failover_unplanned.yml failback.yml reprotect.yml recover_primary.yml dr_test.yml`) for both inventories; lint. Expected: clean.
- [ ] **Step 4:** Review Focus checks by reading: with `only_namespaces: [app-b]` the loop (`dr_namespaces`) never reaches `app-a`; unplanned path `failed_when: false` + warn is present.

### Task 6: URL output and preflight

**Files:**
- Create: `roles/demo_wiki/tasks/urls.yml`
- Modify: `roles/dr_ops/tasks/failover_planned.yml`, `failback.yml`, `bringup.yml` (print the URL of the active site at the end); `roles/demo_wiki/tasks/deploy.yml` (print both URLs); `preflight.yml` (K3s checks)

**Interfaces:**
- Produces: `urls.yml(dr_wiki_site)`: K3s `kubectl -n <ns> get svc wiki -o jsonpath='{.status.loadBalancer.ingress[0].ip}'`, OpenShift `kubectl -n <ns> get route wiki -o jsonpath='{.spec.host}'`; `debug` message `Wiki.js (<site>): <url>`. Retries until the address exists (LB IP can lag).
- Preflight (K3s, only when `demo_wiki_namespaces | length > 0`): assert each site defines `lb_range`; assert no `svclb-*` DaemonSets in `kube-system` (ServiceLB still enabled -> fail with "start K3s with --disable servicelb").

- [ ] **Step 1:** Implement `urls.yml`, include it from `deploy.yml` (both sites) and from the three dr_ops task files with `dr_wiki_site` set to the now-active site.
- [ ] **Step 2:** Add the preflight tasks as a play block using `kubectl`.
- [ ] **Step 3: Verify:** `--syntax-check preflight.yml` and the DR playbooks on both inventories; lint.

### Task 7: Documentation

**Files:**
- Modify: `docs/USER_GUIDE.md` (new section "Wiki.js demo" inside section 4, the two URLs, what to click, K3s ServiceLB prerequisite, OpenShift SCC fallback, the kubectl commands the playbook runs, what is not tested; also fix the section 6 line "kubernetes.core for ... Kubernetes objects" to state the Wiki.js/MetalLB exception), `README.md`, `roles/dr_protect/README.md`, `roles/dr_ops/README.md`, `roles/dr_ops/meta/argument_specs.yml` and `roles/dr_protect/meta/argument_specs.yml` if variables are added there, `CLAUDE.md` (one line under Layout: Wiki.js manifests are kubectl-applied from `roles/demo_wiki`, deliberate exception to kubernetes.core)

- [ ] **Step 1:** Write the sections; every command shown must appear in a task above. State "traffic switch = open the other URL; DNS not managed".
- [ ] **Step 2: Verify:** `uv run yamllint .` and `uv run ansible-lint`; grep docs for `demo_wiki_` variable names and confirm each exists in `group_vars/all/main.yml`.

### Task 8b: Guide v6 (HTML)

**Files:**
- Create: `docs/PowerScale CSM Replication — K3s + OpenShift multi-namespace DR (v6).html` (copy of v5; v5 stays untouched)
- Modify: `README.md:3`, `CLAUDE.md:3`, `docs/USER_GUIDE.md:3` (v5 -> v6 file name)

**Insertion points in the copy** (use the guide's own markup: `h2` header row with a mono step tag, `code-card` blocks, `data-glean-id` on new blocks):
- D1 ("Protect several namespaces..."): sentence that `app-a` runs Wiki.js + PostgreSQL (one `pgdata` PVC) while `app-b`/`app-c` keep the busybox writer; the `kubectl apply` snippets for the Secret, Postgres, Wiki.js and the Service/Route (the commands the role runs).
- D3 (planned failover): the step "open the other site's URL, the edited page is there"; and the data check shows the marker-row `psql` query next to the existing `exec deploy/writer ... tail` lines (lines ~1032-1037 of v5).
- E4 "What changed in v5, and sources" -> new "What changed in v6" with the Wiki.js demo, MetalLB (K3s) / Route (OpenShift), and a "not run on real clusters" note; keep the v5 changes as history.
- Page `<title>`/header and any visible "v5" text -> v6. Do not touch the `saved from url` comment.

- [ ] **Step 1:** `cp` v5 to v6, then edit the v6 copy only.
- [ ] **Step 2: Verify:** `grep -c "v5" <v6 file>` only in the history paragraph; new blocks use existing CSS classes (no new `<style>`); open the file once in a browser (Playwright or Chrome MCP) and screenshot D1 and D3 to confirm rendering.
- [ ] **Step 3:** `grep -rn "(v5)" README.md CLAUDE.md docs/USER_GUIDE.md` returns nothing.

### Task 8: Final verification

- [ ] **Step 1:** Run the full gate from `CLAUDE.md`: `uv run ansible-lint && uv run yamllint . && uv run ansible-playbook tests/test_dellctl_template.yml && uv run ansible-playbook tests/test_wiki_templates.yml` and `(cd roles/common && uv run molecule test)`.
- [ ] **Step 2:** `--syntax-check` of `site.yml preflight.yml` and every DR playbook for BOTH inventories.
- [ ] **Step 3:** `uv run ansible-inventory -i inventory/<k3s|openshift> --host mgmt_vm --playbook-dir .` resolves all new variables.
- [ ] **Step 4:** Invoke `superpowers:verification-before-completion`; report honestly that nothing ran on real clusters and list the `verify` items still open (image env vars/registry, finalize endpoint, NFS UID/root squash for Postgres, SCC behavior, MetalLB version).
