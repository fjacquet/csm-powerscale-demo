# PowerScale CSM replication: Ansible

Automation of `docs/PowerScale CSM Replication — K3s + OpenShift multi-namespace DR (v6).html`.
Section references (K4, D3a ...) match the guide.

## Setup

```bash
uv sync
uv run ansible-galaxy collection install -r requirements.yml -p collections
cp group_vars/all/vault.yml.example group_vars/all/vault.yml   # fill passwords
uv run ansible-vault encrypt group_vars/all/vault.yml
```

Run on the management VM (needs `kubectl`, `helm`, `dellctl`, `repctl`, `yq`, `oc` for OpenShift,
`python3-kubernetes python3-yaml python3-pexpect`, and `pip install isilon-sdk==0.7.0` for the `dellemc.powerscale` modules; `-e tools_install=true` installs all but kubectl/oc).
Pick the track with the inventory: `-i inventory/k3s` (default, in `ansible.cfg`) or `-i inventory/openshift`.
Per-platform data (`platform`, `sites`, `arrays`) lives in `inventory/<track>/group_vars/all/platform.yml`;
shared settings stay in `group_vars/all/main.yml`. A wrong `-e platform=...` is rejected by the `common` role.

## Use

| Goal | Command |
|---|---|
| Read-only health check (run before the demo) | `uv run ansible-playbook preflight.yml` |
| Build everything (section 02 check, K1, K3-K8 or O1-O6, D1/D1a/D2) | `uv run ansible-playbook site.yml --ask-vault-pass` |
| Run one step | `--tags k4` (k1,k3..k8,o1..o6,d1,d1a,d2) |
| Planned failover + bring-up + data check | `failover_planned.yml -e confirm_failover=yes` |
| Reprotect | `reprotect.yml -e confirm_reprotect=yes` |
| Unplanned failover | `failover_unplanned.yml -e confirm_unplanned_failover=yes` |
| Primary back after unplanned | `recover_primary.yml -e confirm_recover=yes` |
| Failback | `failback.yml -e confirm_failback=yes [-e failback_mode=discard -e confirm_discard=yes]` |
| One namespace only | `-e '{"only_namespaces": ["app-b"]}'` |
| Wiki.js demo (app-a) | part of `site.yml` (`--tags demo_wiki,protect`); see USER_GUIDE "Wiki.js demo" |
| Writable snapshots behind PPDM (experiment) | `writable_snapshot_demo.yml --tags preflight,csm` (never part of `site.yml`); see docs/WRITABLE_SNAPSHOT_DEMO.md |
| Cleanup | `cleanup.yml -e confirm_cleanup=yes [-e remove_driver=yes]` |

Failover playbooks are never reachable from `site.yml`. `site.yml` refuses to run while `~/dr/state` says
`secondary-active` for any protected namespace (it would scale primary apps back up). The state is per namespace.

## Important: Basic auth on the arrays

`dellemc.powerscale` (`ps_access: api`, default) uses HTTP Basic auth. The guide's arrays have `auth_basic = false`
(session login only, which the CSI driver uses). `preflight.yml` fails with a clear message until Basic is allowed for
`ps_admin_user`. Alternative without touching the arrays: `-e ps_access=ssh` (raw `isi` over SSH, inventory group `powerscale`).

## Documentation

Full walkthrough, demo script, variable reference and troubleshooting: [docs/USER_GUIDE.md](docs/USER_GUIDE.md).
Which install tool when (dellctl, Helm, Operator): [docs/INSTALL_METHODS.md](docs/INSTALL_METHODS.md).
Writable snapshots behind PPDM backups (experiment, with Mermaid diagrams): [docs/WRITABLE_SNAPSHOT_DEMO.md](docs/WRITABLE_SNAPSHOT_DEMO.md).

## Not automated on purpose

- Section 02 changes to the arrays: assert-only (official `dellemc.powerscale` modules in check mode). `--tags ps_configure` and `--tags synciq_smoke` opt in
  (run the smoke test on both ps1 and ps2). The guide's "step 02b" (csi-user role) has no commands: only checked.
- `oc login`, traffic switch (DNS/Ingress/Route), GitOps pause, VS Code section.

## Quality gates

```bash
uv run ansible-lint && uv run yamllint . && uv run ansible-playbook tests/test_dellctl_template.yml && uv run ansible-playbook tests/test_wiki_templates.yml
```

## Validation status

Linted, syntax-checked and offline-tested only: no cluster or array was available. The guide itself marks Parts C and D
as not lab-run. Every "verify" item of the guide is one variable in `group_vars/all/main.yml`.
