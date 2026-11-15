# csm_k3s

K3 to K9: dellctl configs, generation, manifest split, prerequisites, Helm install, replication wiring and verification (K3s track).

## Requirements

Runs on the management VM. Shared inputs (`sites`, `arrays`, `platform`, namespaces) come from the inventory and
`group_vars/all`; see `docs/USER_GUIDE.md`. Collections: `kubernetes.core`.

## Role variables

| Variable | Type | Default | Description |
|---|---|---|---|
| `chart_version` | str | `1.11.2` | container-storage-modules chart version |
| `controller_count` | int | `1` | Driver controller replicas (1 on single-node K3s) |
| `csm_retention_policy` | str | `Retain` | remote PV/RG retention |
| `allow_pvc_creation_on_target` | bool | `False` | Option A (true) or B (false) of guide section 03c |
| `option_a_values_path` | str | `` | yaml path of the PVC-creation flag in the values file (option A only) |
| `skip_cert_validation` | bool | `True` | Lab only |

## Example playbook

```yaml
- hosts: mgmt
  roles:
    - role: csm_k3s
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
