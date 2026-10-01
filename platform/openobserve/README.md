# OpenObserve

Metrics, logs and traces of the cluster in one backend, at
`https://o2.int.0xc0.cc` (WARP only). Single node, local disk on a Longhorn
volume, 30 days of retention. The collectors in `openobserve-collector` send to
it; what they collect is in their `kustomization.yaml`.

## Before the first sync: the root user

OpenObserve reads its root user from Vault, `platform/shared/openobserve-root`,
which the collectors use too. The operator writes it once, over WARP, with the
password generated straight into stdin (the `vault` repo, README, "Writing or
rotating a secret"):

```sh
export VAULT_ADDR=https://vault.int.0xc0.cc
mise exec -- vault login -no-print
mise exec -- vault kv put -mount=platform shared/openobserve-root email=<your email>
# key=- stores stdin as it is: strip openssl's trailing newline, or it becomes
# part of the password.
openssl rand -base64 24 | tr -d '\n' | mise exec -- vault kv patch -mount=platform shared/openobserve-root password=-
mise exec -- vault kv metadata put -mount=platform \
  -custom-metadata=owner=operator -custom-metadata=rotated_at="$(date +%F)" shared/openobserve-root
rm -f ~/.vault-token
```

To log in, read the password when needed:
`vault kv get -mount=platform -field=password shared/openobserve-root`.

OpenObserve creates its root user from the environment once, on its first
start, and never updates it from there (seen on the first sync, 2026-10-01).
Rotating the password is therefore two steps, in this order:

1. Change it in OpenObserve (the UI, user settings, or its users API), to the
   new value.
2. Write the same value to Vault with `kv patch`. Vault Secrets Operator
   rewrites both Secrets within the hour and restarts OpenObserve and the
   collectors, which then send with it.

Between the two, the collectors are refused (401) and buffer what they can.

## After the first sync, by hand

- **Dashboards:** import the Kubernetes ones from
  [openobserve/dashboards](https://github.com/openobserve/dashboards)
  (Dashboards → Import), and the components' that exist there (Traefik,
  Longhorn, Vault, CrowdSec, etcd). They live in OpenObserve's own store, on
  the volume, not in this repo.
- **Alerts:** the Kubernetes alert rules from the same repository. Their
  destination (mail, Telegram, ntfy) is the operator's choice, still pending.
- **Check every target is up:** the target allocator lists them:

  ```sh
  kubectl -n openobserve-collector port-forward svc/openobserve-collector-gateway-targetallocator 8080:80
  curl -s localhost:8080/jobs | jq 'keys'
  ```
