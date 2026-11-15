# common

Section 01/03b guardrails: validates platform, prefixes, 63-char volume group names and kubeconfigs; creates working dirs and the operator env file.

## Requirements

Runs on the management VM. Shared inputs (`sites`, `arrays`, `platform`, namespaces) come from the inventory and
`group_vars/all`; see `docs/USER_GUIDE.md`. Collections: `kubernetes.core`.

## Role variables

| Variable | Type | Default | Description |
|---|---|---|---|
| `platform` | str | `` | k3s or openshift; must match the inventory |
| `sites` | dict | `` | Cluster definitions keyed primary/secondary (inventory platform.yml) |
| `arrays` | dict | `` | PowerScale arrays keyed ps1/ps2 |
| `app_namespaces` | list | `` | Namespaces protected and failed over |
| `rpo` | str | `Five_Minutes` | SyncIQ RPO used in policy names |
| `dr_dir` | path | `` | Directory for the RG map, state file and logs |
| `common_require_passwords` | bool | `False` | Assert the vault passwords exist |
| `common_create_dirs` | bool | `True` | Create working dirs and the env file |

## Example playbook

```yaml
- hosts: mgmt
  roles:
    - role: common
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
