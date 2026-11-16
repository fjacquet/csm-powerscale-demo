# dr_ops

D3 to D6: planned and unplanned failover, bring-up, reprotect, primary recovery, failback, the data check and replication maintenance (suspend, resume, sync). Use with tasks_from.

## Requirements

Runs on the management VM. Shared inputs (`sites`, `arrays`, `platform`, namespaces) come from the inventory and
`group_vars/all`; see `docs/USER_GUIDE.md`. Collections: `kubernetes.core`, `dellemc.powerscale`.

## Role variables

| Variable | Type | Default | Description |
|---|---|---|---|
| `dr_namespaces` | list | `` | Namespaces to act on (default app_namespaces) |
| `repctl_wait_args` | str | `--wait` | Flag making repctl wait for completion |
| `repctl_create_pvc_args` | str | `--clusters secondary` | Extra args for repctl create pvc (option B) |
| `dr_maint_action` | str | `` | `suspend`, `resume` or `sync` for `tasks_from: maintenance.yml` (`replication_maintenance.yml`) |
| `failback_mode` | str | `keep` | keep or discard writes made on secondary |
| `demo_data_check` | bool | `True` | Capture and verify the last write per namespace (a PostgreSQL marker row for namespaces in `demo_wiki_namespaces`) |

## Example playbook

```yaml
- hosts: mgmt
  tasks:
    - ansible.builtin.import_role:
        name: dr_ops
        tasks_from: failover_planned.yml
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
