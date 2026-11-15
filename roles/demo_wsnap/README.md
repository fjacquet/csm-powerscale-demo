# demo_wsnap

Experiment: observe whether a PPDM 20.3 backup of a PV on PowerScale goes through a OneFS writable snapshot. Two layers: the CSM layer
(snapshot, writable clone, array check, no PPDM) and the PPDM layer (controller ConfigMaps, optional REST run, evidence and verdict).
Walkthrough with diagrams: [docs/WRITABLE_SNAPSHOT_DEMO.md](../../docs/WRITABLE_SNAPSHOT_DEMO.md).

## Requirements

Runs on the management VM (`hosts: mgmt`) against the primary site. Needs `kubectl`, the kubeconfig in `sites.primary`, OneFS 9.13+,
Kubernetes 1.34+, csi-isilon 2.18.0+ with the snapshotter and resizer sidecars, an external snapshot controller v8.4.0+, PPDM 20.3, and
the Wiki.js demo (`pgdata`). Use `writable_snapshot_demo.yml` with tags; the role is never reached by `site.yml`.

## Role variables

- `demo_wsnap_sc_name` (default `powerscale-writable-snapshot`): StorageClass B, writable clone class.
- `demo_wsnap_source_sc` (default empty): StorageClass A of the protected PVC; empty means the single class with `isReplicationEnabled`.
- `demo_wsnap_snapclass` (default empty): VolumeSnapshotClass; empty means the single csi-isilon class, created when none exists.
- `demo_wsnap_protected_ns` (default first Wiki.js namespace), `demo_wsnap_protected_pvc` (default `pgdata`): the replicated PVC that is snapshotted, cloned and protected.
- `demo_wsnap_datamover_ns` (default `powerprotect`): namespace of the PPDM data mover for A and B.
- `demo_wsnap_ppdm_automate` (default `false`): drive PPDM through REST; needs `demo_wsnap_ppdm_host`,
  `demo_wsnap_ppdm_policy` (an existing policy) and the password in `demo_wsnap_ppdm_password` or the environment variable `PPDM_PASSWORD`.
- Lab values, placeholders in `platform.yml`: `demo_wsnap_ppdm_host`, `demo_wsnap_ppdm_policy`, `demo_wsnap_ppdm_datadomain`, `demo_wsnap_k8s_api_fqdn`, `demo_wsnap_k8s_api_port` (6443), `demo_wsnap_ppdm_rbac_dir`, `demo_wsnap_ppdm_sa_ns` (`powerprotect`). `--tags register` registers the cluster and creates the policy (`-e demo_wsnap_ppdm_automate=true -e confirm_ppdm_changes=yes`).
- `demo_wsnap_ppdm_user` (default `admin`), `demo_wsnap_poll_seconds` (3), `demo_wsnap_evidence_timeout` (1800).

Guards: `-e confirm_ppdm_changes=yes` (ConfigMaps and controller restart), `-e confirm_writable_cleanup=yes` (cleanup),
`-e demo_wsnap_remove_sc=yes` (also delete StorageClass B, only when this role created it: it carries the label `demo-wsnap=owner`).

## Example

```bash
ansible-playbook -i inventory/openshift writable_snapshot_demo.yml --tags preflight,csm
ansible-playbook -i inventory/openshift writable_snapshot_demo.yml --tags ppdm,evidence -e confirm_ppdm_changes=yes
```

## Behavior

Idempotency: StorageClasses and ConfigMaps are applied only when missing or changed; the scenario recreates its objects each run.
Check mode: not supported (the role runs `kubectl` and reads the arrays). Rollback: `--tags cleanup`. Provenance: the REST login flow is
adapted from https://github.com/bottkars/ansible_ppdm (commit f66c9fa7246c2b9c1e59140213c41adc3e163bdb). License: Apache-2.0. Author: Fred Jacquet.
