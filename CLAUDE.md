# gitops

What runs in the homelab's Kubernetes cluster, as manifests ArgoCD applies.

- `clusters/prod/` — what ArgoCD deploys. The root `Application`, written by
  the `argocd` role in `infrastructure`, syncs it recursively: one
  `Application` per component (app of apps). ArgoCD itself is installed and
  upgraded by RKE2's helm-controller, never from here.

## CURRENT PHASE: 2 (Cluster)

Phase 2 builds the RKE2 cluster and ArgoCD (workspace `docs/design.md`). The
cluster exists; ArgoCD syncs `clusters/prod/` from `main`.

## What goes where

One RKE2 cluster, in the `platform` zone, holds everything after phase 1:

- **Shared services**: ArgoCD, Vault, Prometheus and Grafana, data services
  (Postgres, Redis).
- **Applications**.

They are separated by namespace and NetworkPolicy. Traffic enters through the
HAProxy load balancer in front of the cluster, and the ingress carries the WAF
(open-appsec).

The network is decided by `../infrastructure/environments/prod/terraform.tfvars`
(`zones`, `vms`, `transit`) and explained in `../infrastructure/docs/zones.md`.

## Hard rules

- Everything written is in English: files, file names, comments, commits,
  branches and PRs.
- No work without an issue on the org project board. The PR links it
  (`Closes #N` / `Refs owner/repo#N`) or the `issue` check fails. See the
  workspace `CLAUDE.md`, section Tracking.
- **Images pinned by digest.** Never `latest`, never a tag alone.
- **The WAF lives at the ingress** (open-appsec). Do not duplicate it in a
  service.
- **Every namespace denies by default** and opens only what it needs, with
  NetworkPolicies. Data services accept connections and initiate none.
- **Every persistent volume declares its backup** in a comment: destination
  and frequency. Without that, the service is not deployed.
- **Secrets:** what the cluster needs to boot comes from SOPS+age; everything
  else from Vault, from phase 3. Never a secret in cleartext in a manifest.
- **Portals** (Grafana, ArgoCD) are published only behind Cloudflare Access.
  **Vault, the Kubernetes API and other admin interfaces are never published**:
  they are reached over WARP.

## Promotion

Applications promote test→prod with **the same digest**. The image is never
rebuilt between environments.

## Before opening a PR

For changes touching exposure or NetworkPolicies, run
`homelab:network-reviewer`.

Render every `Application` you touch with its own values and validate what
comes out, with the tools in `mise.toml`:

```
yq '.spec.source.helm.valuesObject' clusters/prod/<app>.yaml > /tmp/values.yaml
helm template <app> <chart> --version <version> --repo <repo> -n <namespace> \
  --kube-version 1.36.4 -f /tmp/values.yaml \
  | kubeconform -strict -ignore-missing-schemas -kubernetes-version 1.36.0 -summary
```
