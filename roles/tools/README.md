# tools

Checks (default) or installs the CSM tools (helm, dellctl, repctl, yq), adds the Dell Helm repo and registers clusters in dellctl and repctl.

## Requirements

Runs on the management VM. Shared inputs (`sites`, `arrays`, `platform`, namespaces) come from the inventory and
`group_vars/all`; see `docs/USER_GUIDE.md`. Collections: `kubernetes.core`.

## Role variables

| Variable | Type | Default | Description |
|---|---|---|---|
| `tools_install` | bool | `False` | Install dellctl, repctl, yq, Helm 4 and python deps (needs sudo) |
| `tools_check_only` | bool | `False` | Only verify; skip repo add and cluster registration |
| `csm_tools_version` | str | `v1.18.2` | dellctl/repctl release tag |

## Example playbook

```yaml
- hosts: mgmt
  roles:
    - role: tools
```

## Idempotency and check mode

Re-runnable: apply tasks use declarative modules (`kubernetes.core.*`) or are guarded with `creates:`/`removes:`.
Action roles (`dr_ops`) perform failovers and are intentionally not idempotent; they require explicit confirmation in
the calling playbook. Check mode is supported by the declarative tasks; `command`/`shell` tasks do not run changes in it.

## Rollback

Declarative changes are reverted by re-running with the previous values. DR actions are reverted by the matching
playbook (failback, reprotect). StorageClass parameters are immutable: delete and recreate.

## License and author

Apache-2.0. Fred Jacquet.
