# User guide

Automates `docs/PowerScale CSM Replication — K3s + OpenShift multi-namespace DR (v6).html`.
Step names (K4, D3a ...) refer to that guide.

## 1. What it builds

```
management VM (Ansible runs here)
  ├─ K3s or OpenShift "primary"   (cluster ID primary,   array ps1)
  └─ K3s or OpenShift "secondary" (cluster ID secondary, array ps2)
       CSI PowerScale + CSM Replication, one SyncIQ policy and one RG per namespace
```

| Phase | Playbook / tags | Guide steps |
|---|---|---|
| Health check | `preflight.yml` | 01, 02, K1, K2, K7 |
| Arrays | `site.yml` (assert-only); `--tags ps_configure`, `--tags synciq_smoke` opt in | 02 |
| Node prep (K3s) | `--tags k1` | K1 |
| Tools | `--tags tools` (`-e tools_install=true` installs) | K2, K5a |
| Install K3s | `--tags k3,k4,k5,k6,k7,k8` (`smoke` opt-in) | K3-K9 |
| Install OpenShift | `--tags o1,o2,o3,o4,o5,o6` | O1-O6 |
| Protect namespaces | `--tags d1,d1a,d2` | D1, D1a, D2 |
| Planned failover | `failover_planned.yml` | D3a, D3b, D6 |
| Reprotect | `reprotect.yml` | D3c |
| Unplanned failover | `failover_unplanned.yml` | D4 |
| Primary recovered | `recover_primary.yml` | D4a |
| Failback | `failback.yml` | D5 |
| Data check | `dr_test.yml` | D6 |
| Cleanup | `cleanup.yml` | E3 |

Which install tool to use when (dellctl, Helm, Operator): see [INSTALL_METHODS.md](INSTALL_METHODS.md).

## 2. Prerequisites

1. Management VM with `kubectl`, `helm` 4, `dellctl`/`repctl` 1.18.x, `yq` 4, `oc` (OpenShift).
   `-e tools_install=true` installs everything except `kubectl`/`oc`.
2. Python packages for the interpreter Ansible uses: `kubernetes`, `PyYAML`, `pexpect`, `isilon-sdk==0.7.0`.
3. `~/.kube/primary.yaml` and `~/.kube/secondary.yaml` (K3s: copy from the VMs and fix `server:` away from
   127.0.0.1; OpenShift: `oc login --kubeconfig`). Preflight checks both.
4. Section 02 of the guide done on both arrays, including the `csi-user` and role (guide step 02b).
5. Basic authentication allowed on the PowerScale Platform API for `ps_admin_user` (see 7).
6. `uv sync`, then `uv run ansible-galaxy collection install -r requirements.yml -p collections`.
7. Vault file: `cp group_vars/all/vault.yml.example group_vars/all/vault.yml`, fill, `ansible-vault encrypt`.

## 3. Configure

Shared settings are in `group_vars/all/main.yml` (versions, namespaces, RPO, retention). Per-track data is in
`inventory/<k3s|openshift>/group_vars/all/platform.yml` (IPs or API FQDNs, array names, prefixes) and
`inventory/<track>/hosts.yml` (management VM connection, K3s nodes, arrays). Playbooks sit at the repo root so the
shared `group_vars` and the vault load for all of them. Key switches:

| Variable | Meaning |
|---|---|
| `-i inventory/k3s` / `-i inventory/openshift` | selects the track (`platform`, `sites`, `arrays` come from the inventory) |
| `ps_access` | `api` (dellemc.powerscale, needs Basic auth) or `ssh` (raw `isi`) |
| `app_namespaces` | namespaces protected and failed over (one RG each) |
| `sites.*.vg_prefix` | `volumeGroupPrefix`; must differ per cluster |
| `allow_pvc_creation_on_target` | false = option B (`repctl create pvc`), true = option A |
| `csm_retention_policy` | `Retain` (recommended) or `Delete` for replica PV/RG |
| `ps_admin_user` | OneFS user for the API modules (default `ps_user`) |
| `only_namespaces` | `-e '{"only_namespaces":["app-b"]}'` narrows any DR playbook |

Items the guide tags "verify" are single variables: `rg_annotation_key`, `retention_pv_param`, `retention_rg_param`,
`option_a_values_path`, `repctl_create_pvc_args`, `repctl_state_failedover`, `dellctl_password_prompt`.

## 4. Demo script

1. `uv run ansible-playbook preflight.yml --ask-vault-pass` (everything green).
2. `uv run ansible-playbook site.yml --ask-vault-pass` (install + three protected namespaces).
3. Show `kubectl get rg` on both clusters and the SyncIQ policies on ps1.
   Open the Wiki.js URL of the primary (printed by `site.yml`, see "Wiki.js demo" below) and edit a page.
4. `failover_planned.yml -e confirm_failover=yes`: writers stop, RGs fail over, PVCs and apps come up on
   secondary, the data check proves the last write survived. Open the secondary's Wiki.js URL (printed at the end of the
   bring-up): the page you edited is there.
5. `reprotect.yml -e confirm_reprotect=yes`, then `failback.yml -e confirm_failback=yes`.
6. Single app: repeat step 4 with `-e '{"only_namespaces":["app-b"]}'`.
7. Unplanned (maintenance window only): `failover_unplanned.yml`, then `recover_primary.yml`, then reprotect or failback.
8. Traffic switch (DNS/Ingress/Route) and GitOps pause are manual: say so during the demo.

### Wiki.js demo (namespaces in `demo_wiki_namespaces`, default `app-a`)

`app-a` runs Wiki.js + PostgreSQL instead of the busybox writer (`app-b` and `app-c` keep it). The only volume is the
PostgreSQL one (`pgdata`), so the database and the wiki move together. Manifests are rendered to `~/dr/wiki/<site>/` and applied
with `kubectl`; you can read or re-run them yourself:

```bash
kubectl --kubeconfig ~/.kube/primary.yaml apply -f ~/dr/wiki/primary/app-a-pvc.yaml        # primary only
kubectl --kubeconfig ~/.kube/primary.yaml apply -f ~/dr/wiki/primary/app-a-workloads.yaml  # primary only
kubectl --kubeconfig ~/.kube/primary.yaml apply -f ~/dr/wiki/primary/app-a-shared.yaml     # both sites
kubectl --kubeconfig ~/.kube/secondary.yaml apply -f ~/dr/wiki/secondary/app-a-shared.yaml
kubectl --kubeconfig ~/.kube/primary.yaml -n app-a get svc wiki        # K3s: EXTERNAL-IP from MetalLB
kubectl --kubeconfig ~/.kube/primary.yaml -n app-a get route wiki      # OpenShift
```

- The PVC exists on the primary only: on the secondary `repctl create pvc` recreates it from the replica PV at bring-up
  (a PVC already present there would make the bring-up skip that step).
- The Wiki.js and PostgreSQL pods on the secondary are standby copies at 0 replicas; they start at bring-up, PostgreSQL
  possibly after Wiki.js (Wiki.js restarts until the database answers). The data check waits for it (30 x 10 s).
- Consistency proof: before the failover a marker row is written to PostgreSQL (`dr_marker`), after it the row is searched on
  the target. For the audience: the edited page opens at the other site's URL. Traffic switch = open the other URL; DNS is not managed.
- K3s: MetalLB (L2) gives each site an address from `sites.<site>.lb_range` in `inventory/k3s/group_vars/all/platform.yml`
  (replace the placeholders with free addresses). K3s must run with `--disable servicelb`; `preflight.yml` checks it.
  OpenShift: a `Route`, no MetalLB.
- Credentials: `vault_wiki_db_password` (the same on both sites: it lives in the replicated data) and `vault_wiki_admin_password`.
- The setup wizard is finalized by the playbook (`demo_wiki_bootstrap`). If the page asks for setup, run the wizard by hand once on
  the primary; the result replicates. Verify the setup endpoint before a customer demo.
- OpenShift: both images must run under a random UID (sclorg PostgreSQL image, `emptyDir` for Wiki.js data). Only if the lab shows a
  crash, grant `anyuid` on `app-a` (lab only).
- Not run on real clusters: image variables (`demo_wiki_db_image`), PostgreSQL on the PowerScale NFS export (UID, root squash)
  and the MetalLB version are marked `verify` in `group_vars/all/main.yml`.
- Disable with `demo_wiki_namespaces: []` (busybox writers everywhere, no MetalLB).

### Writable snapshots behind PPDM backups (experiment)

Checks whether a PPDM backup of a PV (here `pgdata`) goes through a OneFS writable snapshot, as Dell states. The full walkthrough, with
state, sequence and decision diagrams, is in [WRITABLE_SNAPSHOT_DEMO.md](WRITABLE_SNAPSHOT_DEMO.md).

- Playbook `writable_snapshot_demo.yml` (never called by `site.yml`); role `demo_wsnap`.
- `--tags preflight,csm`: version checks, then snapshot, writable clone and array check, without PPDM.
- `--tags ppdm,evidence`: PPDM ConfigMaps (`-e confirm_ppdm_changes=yes`), then the verdict `OBSERVED` or `NOT OBSERVED` with a Mermaid timeline.
- `--tags cleanup -e confirm_writable_cleanup=yes` removes the scenario.
- Needs OneFS 9.13+, Kubernetes 1.34+, CSM 1.18, PPDM 20.3. Not run on real clusters, arrays or PPDM yet.

### Suspend, resume and check replication on the arrays

`replication_maintenance.yml` runs `repctl --rg <rg> exec -a suspend|resume|sync` on every protected namespace (or `only_namespaces`).
On PowerScale, suspend disables the SyncIQ policy on the source array, resume enables it again, sync starts a SyncIQ job.

```bash
uv run ansible-playbook replication_maintenance.yml -e dr_maint_action=sync
uv run ansible-playbook replication_maintenance.yml -e dr_maint_action=suspend -e confirm_suspend=yes   # RPO grows until resume
uv run ansible-playbook replication_maintenance.yml -e dr_maint_action=resume
```

- While suspended, writers keep writing on the primary but nothing reaches ps2, not even the scheduled copies. Resume, then run
  `sync` to catch up now instead of waiting for the next interval. Run one action at a time: CSM rejects a new action while one is running.
- Do not leave replication suspended before a planned failover: the final sync needs the policy enabled.
- Array-side check (read-only, on the arrays; trust the array if `kubectl get rg` disagrees):

| When | Array | Command | Expect |
|---|---|---|---|
| any time | ps1 | `isi sync policies list`, then `isi sync policies view <policy>` | Enabled: Yes, Last Job State finished |
| after sync | ps1 | `isi sync jobs list`, `isi sync reports list` | a job while it runs, then a new finished report |
| after suspend | ps1 | `isi sync policies view <policy>` | Enabled: No, no new reports |
| any time | ps2 | `isi sync target list`, `isi sync target view <policy>` | Last Job State finished; writes disabled until a failover |

  The policy name is the one `isi sync policies list` shows (CSM names it after the volume group prefix, the target array and the RPO).
  `isi sync reports list` can be filtered by policy: confirm the flag with `isi sync reports list --help` on your OneFS version.

## 5. Safety guards

- Failover, failback, reprotect, recovery, cleanup and replication suspend need an explicit `-e confirm_...=yes`; `site.yml` never calls them.
- `site.yml` refuses to run while `~/dr/state` lists a protected namespace as `secondary-active` (state is per namespace;
  override: `-e allow_rerun_after_failover=true`). `cleanup.yml` clears the state, the RG map and the last-write files.
- `rg-map.txt` (namespace to RG) is written under `~/dr` and copied to `artifacts/` for use when the primary is gone.
- dellctl outputs are renamed to `custom-*` and never regenerated or overwritten on reruns.
- StorageClass parameters are immutable: a changed value fails with an explanation instead of half-applying.
- Credential files are mode 0600 and tasks that touch them use `no_log`.

## 6. Design choices

- `kubernetes.core` for Helm and Kubernetes objects (exception: the Wiki.js demo and MetalLB are deployed with `kubectl apply`, `roles/demo_wiki`); `dellemc.powerscale` for the arrays (OneFS API, check mode as assertion).
- `dellctl` stays a CLI wrapper (Ansible `expect` answers its password prompts). Dell's CLI reference documents no
  password flag or environment variable (checked against the csm-docs sources), and lists only PowerStore for `dellctl install`;
  the guide's PowerScale usage comes from its lab run, so confirm `dellctl install powerscale --help` on your build.
- Still CLI: licenses (`isi license add`) and `isi sync target break` in the smoke test cleanup.

## 7. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Preflight: array rejected Basic auth | Arrays run with `auth_basic = false`; allow Basic for `ps_admin_user`, or use `-e ps_access=ssh` |
| `isilon_sdk`/`isi_sdk` import error | `pip install isilon-sdk==0.7.0` into the Ansible interpreter |
| dellctl step hangs | prompt text differs: set `dellctl_password_prompt` |
| `repctl create pvc` rejects `--clusters` | clear `repctl_create_pvc_args` |
| RG map empty | check `rg_annotation_key` against `kubectl get pvc -o yaml` |
| StorageClass apply fails | parameters immutable: delete the class, re-provision, rerun |
| Pods CrashLoop after failover | volume still mounted: playbooks scale to 0 first; rerun scale down/up |

## 8. Validation status

Lint (`ansible-lint` production profile, `yamllint`), `--syntax-check` and `tests/test_dellctl_template.yml` and `tests/test_wiki_templates.yml` pass.
No cluster or array was available, so no playbook has run end to end. Do a full dry run before a customer demo.
