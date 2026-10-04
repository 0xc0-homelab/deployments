# OpenObserve

Metrics, logs and traces of the cluster in one backend, at
`https://o2.int.0xc0.cc` (WARP only). Single node, local disk on a Longhorn
volume, 30 days of retention. The collectors in `openobserve-collector` send to
it; what they collect is in their `kustomization.yaml`.

## Credentials: the root user and the collectors' service account

Two, both from Vault through Vault Secrets Operator (the `vault` repo, README,
"Secrets: the standard"):

| What | Vault | Read by |
|---|---|---|
| The root user, the operator's login | `platform/openobserve/root` (`email`, `password`) | OpenObserve, which creates the root from it on its first start |
| The collectors' service account | `platform/openobserve-collector/ingest` (`email`, `token`) | the collectors, as basic auth |

**The root user.** OpenObserve creates it from the environment once, on its
first start, and never updates it from there (seen on the first sync,
2026-10-01). Changing its password is therefore two steps, in this order:
first in OpenObserve (the UI, user settings), then the same value in Vault
with `kv patch`. Changing its email means recreating the volume. To log in,
read the password when needed:
`vault kv get -mount=platform -field=password openobserve/root`.

**The service account** is made in OpenObserve (IAM, then Service accounts),
and OpenObserve generates its token. Its token goes to Vault with `read -rs`,
never on the command line:

```sh
export VAULT_ADDR=https://vault.int.0xc0.cc
mise exec -- vault login -no-print
read -rs t && printf '%s' "$t" | mise exec -- vault kv put -mount=platform openobserve-collector/ingest email=collector@0xc0.cc token=- ; unset t
mise exec -- vault kv metadata put -mount=platform \
  -custom-metadata=owner=operator -custom-metadata=rotated_at="$(date +%F)" openobserve-collector/ingest
rm -f ~/.vault-token
```

Rotating it is a new token in OpenObserve, the same `kv put`, then the old
token revoked there. Vault Secrets Operator rewrites the collectors' Secret
within the hour and restarts them. Changing the root user's password does not
touch ingestion.

## Dashboards: from git

Every dashboard is a JSON file in `dashboards/`, imported into the `0xc0`
folder by a PostSync Job after every sync (`dashboards.yaml`,
`dashboards/import.py`): updated by title, created when new, and deleted from
the folder when its file is gone. Git wins: an edit made in the UI to a
dashboard in `0xc0` is overwritten on the next sync. Dashboards in other
folders are left alone, so the UI stays free for trying things.

- **From the community repository**
  ([openobserve/dashboards](https://github.com/openobserve/dashboards)): the
  Kubernetes ones built for this collector (overview, nodes, node pressure,
  namespaces, pods, events), host metrics, ArgoCD and traces. Node pressure is
  rewritten on kube-state-metrics' `kube_node_status_condition`: its own
  queries need the collector's k8s_cluster receiver, which this chart does not
  run. Left out, because they do not work here:
  - the API server and "Compute Resources" ones, built on
    kube-prometheus-stack's recording rules;
  - "Namespace (Objects)", built on the object watches the collector does not
    run;
  - OpenObserve's "Internals" and "Infrastructure", built for a cluster
    (querier, ingester pods). Its own `zo_*` metrics are scraped
    (`ZO_PROMETHEUS_ENABLED`, the chart's ServiceMonitor) into one of ours
    instead.
- **Written here**, for what the repository has nothing for, or nothing
  that works here: the Kubernetes API server, Traefik, Longhorn, Vault,
  CrowdSec, etcd and OpenObserve itself. PromQL over what the collector
  scrapes; every query was checked against OpenObserve before it was
  committed.

**Changing or adding one:** edit it in the UI in any folder but `0xc0`,
export it (dashboard → settings → export JSON), save it here as
`dashboards/<name>.json`, one-space indented, add it to `configMapGenerator`
in `kustomization.yaml`, and open a PR. The title is its key: renaming it is a
delete and a create.

The ConfigMap holds every dashboard, and a ConfigMap's limit is 1 MiB: about
700 KiB today, most of it the cluster overview. Past roughly 900 KiB, split
the files into a second ConfigMap mounted next to the first.

## Alerts

There are none (operator decision, 2026-10-02). If any are added, mail can go
to sergio@0xc0.cc, which Cloudflare Email Routing forwards.

## Checking every target is up

The target allocator lists them:

```sh
kubectl -n openobserve-collector port-forward deploy/openobserve-collector-gateway-targetallocator 8080:8080
curl -s localhost:8080/jobs | jq 'keys'
```
