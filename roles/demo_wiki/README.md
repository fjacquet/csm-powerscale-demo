# demo_wiki

Wiki.js + PostgreSQL as the DR demo workload for the namespaces in `demo_wiki_namespaces` (default `app-a`), deployed with
`kubectl apply`. On K3s, MetalLB gives each site a LoadBalancer address; on OpenShift a Route is used.

## Requirements

Runs on the management VM. Needs `kubectl`, the kubeconfigs in `sites`, and the vault variables `vault_wiki_db_password` and
`vault_wiki_admin_password`. K3s must run without ServiceLB (`--disable servicelb`). Shared inputs come from
`group_vars/all/main.yml`; the K3s MetalLB ranges are `sites.<site>.lb_range` in the inventory.

## Role variables

| Variable | Type | Default | Description |
|---|---|---|---|
| `demo_wiki_namespaces` | list | `[app-a]` | Namespaces running Wiki.js (`[]` disables the role) |
| `demo_wiki_image` | str | `ghcr.io/requarks/wiki:2` | Wiki.js image |
| `demo_wiki_db_image` | str | `quay.io/sclorg/postgresql-16-c9s:latest` | PostgreSQL image (verify tag) |
| `demo_wiki_db_user` / `demo_wiki_db_name` | str | `wiki` | Database user and name |
| `demo_wiki_db_password` | str | vault | One value for both sites |
| `demo_wiki_admin_email` / `demo_wiki_admin_password` | str | `admin@example.com` / vault | Setup wizard bootstrap |
| `demo_wiki_bootstrap` | bool | `true` | Finalize the setup wizard automatically |
| `demo_wiki_metallb_manifest_url` | str | pinned | MetalLB manifest (K3s only) |

## Example playbook

```yaml
- hosts: mgmt
  roles:
    - role: demo_wiki
```

## Idempotency and check mode

`kubectl apply` is idempotent (`changed` only on `created`/`configured`). The `pgdata` PVC is applied on the primary only.
Check mode is not supported by the `kubectl` tasks.

## Rollback

Delete the namespace objects with `kubectl delete -f ~/dr/wiki/<site>/<file>` or run `cleanup.yml`. The PVC and replication
objects follow the guide's cleanup.

## License and author

Apache-2.0. Fred Jacquet.
