# Wiki.js demo workload — design

Date: 2026-11-14. Status: draft, awaiting approval. Scope: Ansible roles + docs, K3s and OpenShift tracks.

## Intent and why

The DR demo proves consistency with a busybox writer appending a timestamp to `/data/<ns>.log`. Correct but invisible
to a customer. Replace it, for one namespace, with Wiki.js on PostgreSQL: the audience edits a page on the primary
site, the failover runs, and the same page is open in a browser on the secondary site. Persistent volumes come for
free (the database volume), and consistency is provable twice: visually (the page) and automatically (a marker row).

## Goals

- `app-a` runs Wiki.js + PostgreSQL on both tracks; one PVC (`pgdata`), so 1 namespace = 1 RG = 1 PVC still holds
  and DB state moves as a unit.
- Same playbooks drive the demo: `site.yml` deploys, `failover_planned.yml` / `failback.yml` / `reprotect.yml` move it.
  No new DR playbook.
- Each site exposes its own URL: K3s via MetalLB `LoadBalancer` IP (L2 pool per site in `platform.yml`), OpenShift via
  a `Route`. Playbook output prints both URLs. DNS is out of scope: the presenter switches the URL by hand.
- Automated check: marker row written to Postgres before failover, read back on the target site. Replaces the log
  check for `app-a` only.
- Docs: new "Wiki.js demo" part in `docs/USER_GUIDE.md` (step 4 of the demo script, with the two URLs and what to
  click), updates to `README.md`, `dr_protect` and `dr_ops` READMEs, `argument_specs.yml`.

## Non-goals

- No DNS, GSLB, Ingress controller setup, GitOps tooling.
- No Wiki.js tuning, SSO, backup of the wiki, or HA Postgres.
- No change to `app-b` / `app-c`: they keep the busybox writer and the log check.
- No Helm chart for the app.

## Anti-goals (failure even if it "works")

- A change that makes `site.yml` reach a DR playbook, or weakens a `confirm_...=yes` guard.
- Per-track data outside `inventory/<t>/group_vars/all/platform.yml`; `platform`/`sites`/`arrays` in the shared file.
- A demo that only works if the presenter pre-seeds a page by hand with no automated fallback.

## Constraints (from the repo)

- Playbooks stay at repo root; role variables carry the role prefix; Jinja only at the end of task names.
- `dr_protect/standby_kind.yml` copies only Deployments/StatefulSets (replicas 0, annotation `csm-demo/primary-replicas`).
  It does NOT copy Secrets, ConfigMaps, Services or Routes: the new code must create these on both sites itself.
- `dr_ops/scale_up_kind.yml` scales Deployments and StatefulSets up from that annotation. Startup order is not
  controlled: Wiki.js must tolerate Postgres coming up later (readiness / restart is acceptable, document it).
- StorageClass parameters are immutable; the PVC uses the existing `dr_protect_sc`.
- K3s is not installed by this repo (`k3s_nodes` only prepares NFS). K3s ServiceLB (klipper) conflicts with MetalLB:
  disabling it (`--disable servicelb`) is a documented prerequisite, checked by `preflight.yml`, not done by us.
- Nothing has run against real clusters. Everything here is validated by lint, syntax checks and a golden/unit test only.

## Approaches considered

- A: Wiki.js everywhere (`app-a/b/c`). Rejected: 3x Postgres start-up time during failover, no extra story.
- **B (chosen): Wiki.js on `app-a` only, busybox kept on `app-b`/`app-c`.** Selected by variable
  `demo_wiki_namespaces: [app-a]`. Audience sees a browser, multi-namespace failover stays cheap, single-app
  failover (`only_namespaces`) still demos with `app-b`.
- C: a 4th namespace `wiki`. Rejected: breaks the "app-a/b/c" reading of the guide v5, changes shared vars.

Other calls (builder's to confirm, mine not the user's): plain manifests via `kubernetes.core.k8s`, no Helm;
PostgreSQL over MariaDB (Wiki.js's recommended DB — confirm in its docs); MetalLB in L2 mode.

## Design

1. **Variable** `demo_wiki_namespaces` (default `[app-a]`) in `group_vars/all/main.yml`, with `demo_wiki_image`,
   `demo_wiki_db_image`, per-track DB image if needed. `protect_ns.yml` creates the busybox writer only when
   `ns not in demo_wiki_namespaces`; otherwise includes the wiki tasks. Writer deployment name stays
   `demo_writer_deploy` for the other namespaces.
2. **Wiki tasks (primary, both tracks):** Secret (DB credentials + Wiki.js admin), PVC `pgdata` (RWX per existing
   convention), Postgres (StatefulSet, 1 replica, `PGDATA` in a sub-directory of the mount), Wiki.js Deployment
   (`DB_TYPE=postgres`, `DB_HOST` = Postgres Service), Services. Namespace-scoped objects not copied by
   `standby_kind` (Secret, Services, Route/LB) are applied to both clusters by the same tasks, loop over `site_list`;
   workloads exist only on the primary, standby copies come from `standby_kind` as today.
3. **Exposure:** K3s: Service type `LoadBalancer`, address from a per-site MetalLB pool; the MetalLB install
   (helm, `IPAddressPool` + `L2Advertisement`) is a task block run from `csm_k3s` or a small `demo_wiki`/`metallb`
   role (builder decides, mirroring existing structure). OpenShift: `Route` (edge TLS, default hostname per cluster),
   no MetalLB. Pool ranges live in `inventory/k3s/group_vars/all/platform.yml`.
4. **Wizard bootstrap:** Wiki.js 2 starts on a setup wizard. The role finalizes it non-interactively (admin
   account, site URL) so the demo opens on a ready wiki, and seeds one page. To be verified: the finalize endpoint
   and the exact request shape against Wiki.js 2 docs/source.
5. **Consistency check:** before failover, `dr_ops/capture_last_write.yml` branches on namespace: for wiki
   namespaces it inserts/reads a marker row (timestamp) in Postgres via `kubectl exec ... psql`; `check_data_ns.yml`
   searches the row on the target site. Stored under `~/dr/last-<ns>.txt` as now. Planned failover scales down first
   (clean Postgres shutdown); unplanned relies on WAL crash recovery.
6. **Output:** end of `site.yml`, `failover_planned.yml`, `failback.yml` print the Wiki.js URL of the active site.
   `preflight.yml` checks MetalLB pool defined (K3s) and ServiceLB disabled.
7. **Docs:** as in Goals. States plainly: "traffic switch = open the other URL".

## Testing

- `ansible-lint`, `yamllint`, `--syntax-check` on both inventories, `ansible-inventory --host mgmt_vm` resolves new vars.
- Extend `tests/` with a render test for the new manifests (like `test_dellctl_template.yml`).
- Real-world validation is a lab run; documented as not done.

## Open risks (verify before or during build)

- PostgreSQL on PowerScale NFS: UID/GID mapping and root squash on the CSM-created export; `PGDATA` sub-directory needed.
- OpenShift SCC: upstream `postgres` image (UID 999) fails under random UID; use the OpenShift-compatible image
  (e.g. sclorg/Red Hat PostgreSQL). Wiki.js image under restricted-v2 SCC: upstream ships `dev/openshift` Dockerfile.
- Wiki.js finalize automation and `DB_PASS_FILE` support are not confirmed from official docs.
- MetalLB vs K3s ServiceLB; pool addresses must be free on the lab network.

## Settled after builder review

Answered by the repo:
- **Standby PVC.** The secondary PVC is never created by the app: `dr_ops/bringup_ns.yml` runs `repctl create pvc` from the
  replica PV (option B), then waits for Bound, then `scale_up`. So Postgres is a **Deployment (strategy Recreate) that
  references the pre-created PVC `pgdata` by `claimName`, no `volumeClaimTemplates`** (a template would create an empty PVC
  on failover). Same pattern as the busybox writer.
- **Re-run safety.** Wiki objects are applied from `protect_ns.yml` (called by `site.yml`), which `~/dr/state` already
  blocks after a failover. No new playbook, no `confirm_...` change. Secrets/Services stay on both sites after failback.
- **Shared password.** The DB password lives in the replicated data, so the Secret must be identical on both sites:
  one vault variable, never generated per site.

My calls (builder: keep unless the lab says otherwise):
- **Check ordering.** `capture_last_write` runs on the primary before scale-down (as today). The check on the target runs after
  `scale_up` (which waits on rollout) and retries `psql` with an `until` loop until Postgres answers (e.g. 30 x 10 s).
- **Wizard.** Try the Wiki.js 2 finalize endpoint first. If it is not usable, fallback = a documented one-time manual wizard
  on the primary before the demo (the data then replicates). No direct SQL seeding.
- **Site URL.** Local authentication only; the site URL stored in the replicated DB may point at the primary after failover
  and is accepted and documented (it only matters for OAuth callbacks).
- **MetalLB.** Helm chart from the management VM (internet assumed, as for CSM); pool ranges are placeholders in
  `platform.yml` that the user must set to free lab addresses.
- **Images (user chose option 1: OpenShift-compatible).** ONE Postgres image on both tracks: the sclorg/Red Hat PostgreSQL
  image (runs under OpenShift's random UID and on plain K3s), so K3s and OpenShift share one manifest and the standby copy
  never diverges. Env vars `POSTGRESQL_USER/PASSWORD/DATABASE`, data dir `/var/lib/pgsql/data` (to confirm in the image
  README for the chosen version; registry choice quay.io vs registry.redhat.io decided by pull-secret needs). Wiki.js: upstream
  `ghcr.io/requarks/wiki:2` (runs as user `wiki`); an `emptyDir` is mounted on its writable `dataPath` (default `./data`,
  overridable in `config.yml`) so a random UID can write; content lives in Postgres. Fallback only if the lab shows a crash:
  `anyuid` SCC on `app-a`, lab-only, documented. No custom image build.

## Amendment (user, 2026-11-14): kubectl-first deployment

Deploy as much as possible with `kubectl`, not `kubernetes.core.k8s`. My reading, applied unless corrected:
- Wiki.js manifests are Jinja templates in the role's `templates/` (one YAML per object group). Ansible renders them to
  `{{ dr_dir }}/wiki/<site>/` and runs `kubectl --kubeconfig <site> apply -f <file>`. Plain `.yaml` files on disk means the
  presenter or a customer can re-run or read the same `kubectl apply -f` / `kubectl get` commands, and the USER_GUIDE shows them.
- MetalLB (K3s): `kubectl apply -f` of the pinned upstream `metallb-native.yaml`, then an `IPAddressPool` + `L2Advertisement`
  manifest from the template; no Helm.
- OpenShift Route: `kubectl apply` of a `Route` manifest (`oc` not required).
- `changed_when` derives from kubectl's `created` / `configured` output; `unchanged` = idempotent. Check mode is not
  supported by these tasks (documented, as for other command tasks). Waits use `kubectl rollout status`, `kubectl wait`.
- Marker-row check and URL output already use `kubectl exec` / `kubectl get svc|route`.
- Exception to the USER_GUIDE "kubernetes.core for Kubernetes objects" rule, scoped to the Wiki.js demo and MetalLB; the
  rule and its README line are updated to say so. Existing busybox/other tasks are not rewritten.
- Hard rule above still applies: the `pgdata` manifest is rendered and applied for the primary only; a test asserts the
  secondary rendering has no `PersistentVolumeClaim`.

## Hard rule: the PVC exists on the primary only

`dr_ops/bringup_ns.yml` skips `repctl create pvc` when any PVC already exists on the secondary
(`allow_pvc_creation_on_target: false`, option B, the default in `group_vars/all/main.yml`). If `pgdata` were applied to both
sites, bringup would skip the restore and the secondary would get an empty volume (possibly a new RG in the reverse direction).
So: `pgdata` is created on the primary only. Only Secret, Service and Route/LB go to both sites. Test: render the
secondary-side manifests and assert they contain no `PersistentVolumeClaim`.

## Left to the builder

File layout (role vs tasks in `csm_k3s`/`dr_protect`), exact image tags, resource requests, probes, the Postgres
password handling (vault variable vs generated), and the test shape.
