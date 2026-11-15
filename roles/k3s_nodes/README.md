# k3s_nodes

K1: NFS client and reachability checks on each K3s node (runs on the k3s inventory group).

## Requirements

Runs on the management VM. Shared inputs (`sites`, `arrays`, `platform`, namespaces) come from the inventory and
`group_vars/all`; see `docs/USER_GUIDE.md`. Collections: `kubernetes.core`.

## Role variables

| Variable | Type | Default | Description |
|---|---|---|---|
| `k3s_nodes_check_only` | bool | `False` | Skip package installation |

## Example playbook

```yaml
- hosts: mgmt
  roles:
    - role: k3s_nodes
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
