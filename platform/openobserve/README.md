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

## Dashboards: from git

Every dashboard is a JSON file in `dashboards/`, imported into the `homelab`
folder by a PostSync Job after every sync (`dashboards.yaml`,
`dashboards/import.py`): updated by title, created when new, and deleted from
the folder when its file is gone. Git wins: an edit made in the UI to a
dashboard in `homelab` is overwritten on the next sync. Dashboards in other
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
- **The homelab's own**, for what the repository has nothing for, or nothing
  that works here: the Kubernetes API server, Traefik, Longhorn, Vault,
  CrowdSec, etcd and OpenObserve itself. PromQL over what the collector scrapes; every query was
  checked against OpenObserve before it was committed.

**Changing or adding one:** edit it in the UI in any folder but `homelab`,
export it (dashboard → settings → export JSON), save it here as
`dashboards/<name>.json`, one-space indented, add it to `configMapGenerator`
in `kustomization.yaml`, and open a PR. The title is its key: renaming it is a
delete and a create.

The ConfigMap holds every dashboard, and a ConfigMap's limit is 1 MiB: about
700 KiB today, most of it the cluster overview. Past roughly 900 KiB, split
the files into a second ConfigMap mounted next to the first.

## Alerts

None yet (operator decision, 2026-10-02). When they come, mail can go to
sergio@0xc0.cc, which Cloudflare Email Routing forwards (infrastructure#157).

## Checking every target is up

The target allocator lists them:

```sh
kubectl -n openobserve-collector port-forward deploy/openobserve-collector-gateway-targetallocator 8080:8080
curl -s localhost:8080/jobs | jq 'keys'
```
