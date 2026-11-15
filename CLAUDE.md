# csm-powerscale-demo

Ansible automation of `docs/PowerScale CSM Replication ... (v6).html` (K3s + OpenShift, multi-namespace DR).
Step names (K4, D3a ...) refer to that guide. Docs: `docs/USER_GUIDE.md`, `README.md`, per-role `README.md`.

## Commands
```bash
uv sync && uv run ansible-galaxy collection install -r requirements.yml -p collections
uv run ansible-lint && uv run yamllint . && uv run ansible-playbook tests/test_dellctl_template.yml
(cd roles/common && uv run molecule test)             # only role testable without clusters/arrays
uv run ansible-playbook -i inventory/<k3s|openshift> --syntax-check <playbook>   # run for BOTH inventories
uv run ansible-inventory -i inventory/k3s --host mgmt_vm --playbook-dir .         # confirm vars resolve
```

## Layout
- Playbooks live at the repo ROOT. group_vars load only beside the inventory or the playbook; a playbook in a
  subdirectory silently loses sites/platform/dr_dir (`--syntax-check` does not catch it).
- Track = inventory (`inventory/k3s`, `inventory/openshift`). Per-track data only in
  `inventory/<t>/group_vars/all/platform.yml`; shared settings in `group_vars/all/main.yml`. Never put
  `platform`/`sites`/`arrays` in the shared file (playbook-dir vars win over inventory vars).
- Wiki.js demo (`roles/demo_wiki`, namespaces in `demo_wiki_namespaces`) and MetalLB are applied with `kubectl apply` of rendered
  manifests, a deliberate exception to kubernetes.core. The `pgdata` PVC is primary-only (bringup skips `repctl create pvc` otherwise).
- Roles: common, tools, k3s_nodes, ps_prep, csm_k3s, csm_ocp, dr_protect, dr_ops. Variables set inside a role need the
  role prefix (ansible-lint var-naming). Task names: Jinja only at the end.

## Gotchas
- DR playbooks need explicit `-e confirm_...=yes`; `site.yml` must never reach them. `~/dr/state` blocks re-running
  site.yml after a failover.
- `dellctl install powerscale` has no documented non-interactive password input: driven by `ansible.builtin.expect`;
  `creates:` protects the `custom-*` files you edit.
- `dellemc.powerscale` needs HTTP Basic; the guide's arrays run auth_basic=false. `ps_access: api|ssh` switches
  (ssh = raw `isi`).
- StorageClass parameters are immutable: retention is set in the files BEFORE the first apply.
- Guide "verify" items are single variables in `group_vars/all/main.yml`.

## Workflow
- Ansible work: use the ansible skills (ansible-docs, ansible-good-practices, ansible-new-role, ansible-new-molecule,
  ansible-zen) instead of writing from memory; validate vendor CLI/module usage before writing it.
- Run superpowers:verification-before-completion before saying "done".

## Status
Lint, syntax, golden test, offline `tests/test_writable_snap_*.yml` and the `common` Molecule scenario pass. Nothing has run
against real clusters, arrays or PPDM: dry-run before any customer demo. `roles/demo_wsnap` (writable snapshots behind PPDM
backups) is an experiment: see `docs/WRITABLE_SNAPSHOT_DEMO.md` and its design spec for what is still unverified.
Vendor PDFs and OpenAPI specs (`docs/references`, `docs/openapi`) are versioned; `test_writable_snap_endpoints` reads them.
