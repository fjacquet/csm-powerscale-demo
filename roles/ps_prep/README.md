# ps_prep

Section 02: verifies (or with tags configures) both PowerScale arrays through dellemc.powerscale (ps_access=api) or raw isi over SSH (ps_access=ssh).

## Requirements

Runs on the management VM. Shared inputs (`sites`, `arrays`, `platform`, namespaces) come from the inventory and
`group_vars/all`; see `docs/USER_GUIDE.md`. Collections: `kubernetes.core`, `dellemc.powerscale`.

## Role variables

| Variable | Type | Default | Description |
|---|---|---|---|
| `ps_access` | str | `api` | Array access method |
| `ps_user` | str | `csi-user` | CSI user checked on the arrays |
| `ps_admin_user` | str | `` | User for the API modules (default ps_user) |
| `isi_path` | path | `/ifs/data/csi` | CSI base path that must exist |
| `skip_cert_validation` | bool | `True` | Do not verify the OneFS certificate (lab only) |

## Example playbook

```yaml
- hosts: mgmt
  roles:
    - role: ps_prep
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
