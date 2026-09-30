# gitops

What runs in the homelab's Kubernetes cluster, as manifests ArgoCD applies.

- `bootstrap/prod/` — what the root `Application` syncs; the `argocd` role in
  `infrastructure` writes that root. It holds the `AppProject`s and the
  `platform` ApplicationSet, which turns every directory under `platform/`
  into an `Application` of its name, in the `platform` project.
- `platform/<component>/` — one Kustomize component per shared service: a
  `kustomization.yaml` whose `helmCharts` inflate 1..n charts, each pinned by
  version with its `values*.yaml`; `resources` for what the charts do not bring
  (its `Namespace`, with its Pod Security and Gateway labels, and its
  NetworkPolicies); `patches` for tweaks. Adding a component is adding its
  directory. Its Application's destination namespace is the directory's name:
  Argo CD puts there what a chart leaves without one, because Kustomize's
  `namespace` field does not reach what `helmCharts` inflates. A component holding data or CRDs sets `Prune=false` through
  `commonAnnotations`.
- ArgoCD itself is installed and upgraded by RKE2's helm-controller, never
  from here. The applications of phase 6 get `apps/`, the same pattern under
  their own `AppProject`.

Operator decision, 2026-09-29 (#16): an ApplicationSet over Kustomize
components, instead of hand-written `Application`s with inline values.

## CURRENT PHASE: 2 (Cluster)

Phase 2 builds the RKE2 cluster and ArgoCD (workspace `docs/design.md`). The
cluster exists; ArgoCD syncs `bootstrap/prod/`, and through it `platform/`,
from `main`.

## What goes where

One RKE2 cluster, in the `platform` zone, holds everything after phase 1:

- **Shared services**: ArgoCD, Vault, Prometheus and Grafana, data services
  (Postgres, Redis).
- **Applications**.

They are separated by namespace and NetworkPolicy. Traffic enters through the
HAProxy load balancer in front of the cluster (layer 4), and the ingress,
Traefik, carries the WAF: CrowdSec's bouncer.

The network is decided by `../infrastructure/environments/prod/terraform.tfvars`
(`zones`, `vms`, `transit`) and explained in `../infrastructure/docs/zones.md`.

## Hard rules

- Everything written is in English: files, file names, comments, commits,
  branches and PRs.
- No work without an issue on the org project board. The PR links it
  (`Closes #N` / `Refs owner/repo#N`) or the `issue` check fails. See the
  workspace `CLAUDE.md`, section Tracking.
- **Images pinned by digest.** Never `latest`, never a tag alone.
- **TLS ends at Traefik**, with a Let's Encrypt wildcard per domain from
  cert-manager (`platform/traefik/certificates.yaml`), on the HTTPS listeners
  of the Gateway. Routes attach to those, never to `web`, which only
  redirects. Adding a domain: its Certificate, its two listeners (`*.d` and
  `d`), and its zone in the Cloudflare token.
- **A name is public only by annotation**: external-dns publishes the
  HTTPRoutes marked `gateway.0xc0.cc/public: "true"`, and only in the domains
  of its `domainFilters` (the zones in the public tunnel's Cloudflare account,
  also listed in infrastructure's `public_domains`). A portal never carries
  it.
- **Two paths in**, separated by network. Public and WARP traffic reaches the
  public VIP (`10.10.4.10`) and `websecure`. The WARP-only path reaches the
  internal VIP (`10.10.4.9`) and Traefik's `internal` entrypoint, which the
  public tunnel never targets. An internal service attaches to the Gateway's
  `internal` listener (`sectionName: internal`) with a `*.int.0xc0.cc` name,
  its namespace labelled `gateway.0xc0.cc/internal: "true"`, and never
  carries the `public` annotation. external-dns never publishes under
  `int.0xc0.cc`. A `*.int.0xc0.cc` name never goes on a `websecure` route
  either: the public `*.0xc0.cc` listener would accept it. The internal path
  has no WAF: CrowdSec's bouncer is on `websecure` only.
- **The WAF lives at the ingress**: CrowdSec's bouncer on Traefik's
  entrypoint. Do not duplicate it in a service. Behind Cloudflare, the client
  is `CF-Connecting-IP`, trusted only from `platform`.
- **Every namespace denies by default** and opens only what it needs, with
  NetworkPolicies. Data services accept connections and initiate none.
- **Every persistent volume declares its backup** in a comment: destination
  and frequency. Without that, the service is not deployed.
- **Secrets:** what the cluster needs to boot comes from SOPS+age; everything
  else from Vault, from phase 3. Never a secret in cleartext in a manifest.
  A component gets its secrets through Vault Secrets Operator: its own
  `VaultAuth` (Kubernetes auth, role named after its namespace, defined in the
  `vault` repo) and a `VaultStaticSecret` or `VaultDynamicSecret` per secret.
  It reads only `platform/<namespace>/*` (an application, `apps/<namespace>/*`).
  Paths are kebab-case, keys inside snake_case.
- **Portals** (Grafana, ArgoCD) are internal: reached only over WARP, never
  published (operator decision, 2026-09-29). **Vault, the Kubernetes API and
  other admin interfaces are never published** either.

## Promotion

Applications promote test→prod with **the same digest**. The image is never
rebuilt between environments.

## Before opening a PR

For changes touching exposure or NetworkPolicies, run
`homelab:network-reviewer`.

Render every component you touch exactly as Argo CD does, and validate what
comes out, with the tools in `mise.toml` (CI runs the same on every PR):

```
kustomize build --enable-helm platform/<component> \
  | kubeconform -strict -ignore-missing-schemas -kubernetes-version 1.36.0 -summary
```

`kustomize` downloads the charts into `platform/<component>/charts/`, which
git ignores: nothing is vendored.
