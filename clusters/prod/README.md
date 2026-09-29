# clusters/prod

What ArgoCD deploys in the homelab's RKE2 cluster. The root `Application`
syncs this directory recursively, pruning and self-healing, from `main`.

ArgoCD itself is not here: RKE2's helm-controller installs and upgrades it,
from the manifests the `argocd` role in `infrastructure` writes on every
server. ArgoCD never manages itself.

Each component gets its own `Application` in this directory (app of apps),
pointing at its chart or its manifests, pinned by version and images by
digest. Nothing here holds a secret, not even encrypted: what the cluster
needs to boot comes from SOPS through Ansible, and everything else from Vault,
from phase 3.

Empty until the first component, the ingress with open-appsec (gitops#6).
