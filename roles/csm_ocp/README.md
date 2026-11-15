# csm_ocp

O1 to O6: CSM Operator, repctl inject, driver secrets, ContainerStorageModule, StorageClass pair and verification (OpenShift track).

## Requirements

Runs on the management VM. Shared inputs (`sites`, `arrays`, `platform`, namespaces) come from the inventory and
`group_vars/all`; see `docs/USER_GUIDE.md`. Collections: `kubernetes.core`.

## Role variables

| Variable | Type | Default | Description |
|---|---|---|---|
| `csm_version` | str | `v1.18.2` | ContainerStorageModule spec.version |
| `ocp_operator_channel` | str | `stable` | OperatorHub channel |
| `ocp_operator_namespace` | str | `openshift-operators` | Operator namespace |
| `ocp_inject_use_sa` | bool | `False` | repctl cluster inject --use-sa after the controller exists |
| `ocp_remote_api_ips` | dict | `{}` | O3a: site id to IP of the OTHER cluster API |
| `allow_pvc_creation_on_target` | bool | `False` | REPLICATION_ALLOW_PVC_CREATION_ON_TARGET |

## Example playbook

```yaml
- hosts: mgmt
  roles:
    - role: csm_ocp
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
