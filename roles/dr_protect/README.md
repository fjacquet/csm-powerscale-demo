# dr_protect

D1, D1a, D2: demo workloads per namespace, namespace to RG map, SyncIQ policy check and standby Deployments on the failover cluster.

## Requirements

Runs on the management VM. Shared inputs (`sites`, `arrays`, `platform`, namespaces) come from the inventory and
`group_vars/all`; see `docs/USER_GUIDE.md`. Collections: `kubernetes.core`, `dellemc.powerscale`.

## Role variables

| Variable | Type | Default | Description |
|---|---|---|---|
| `replication_sc_name` | str | `` | Override the auto-detected replication StorageClass |
| `allow_rerun_after_failover` | bool | `False` | Allow re-running while secondary is the active site |
| `rg_annotation_key` | str | `replication.storage.dell.com/replicationGroupName` | PVC annotation holding the RG name (guide: verify) |
| `demo_pvc_size` | str | `5Gi` | Demo PVC size |
| `demo_image` | str | `busybox:1.36` | Demo writer image |
| `demo_wiki_namespaces` | list | `[app-a]` | Namespaces that run Wiki.js (role `demo_wiki`) instead of the writer |

## Example playbook

```yaml
- hosts: mgmt
  roles:
    - role: dr_protect
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
