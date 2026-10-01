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
openssl rand -base64 24 | mise exec -- vault kv patch -mount=platform shared/openobserve-root password=-
mise exec -- vault kv metadata put -mount=platform \
  -custom-metadata=owner=operator -custom-metadata=rotated_at="$(date +%F)" shared/openobserve-root
rm -f ~/.vault-token
```

To log in, read the password when needed:
`vault kv get -mount=platform -field=password shared/openobserve-root`.

Rotating it is a `kv patch` of `password`: Vault Secrets Operator rewrites both
Secrets within the hour and restarts OpenObserve and the collectors. Whether
OpenObserve also updates an existing root user's password from the
environment is to be confirmed on the first rotation: if the collectors are
refused afterwards (401 in their logs), change it in the UI too.

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
