# offby1-cc

The offby1.cc landing page: the Node server of 0xc0-homelab/offby1.cc, public
at `https://offby1.cc`; `www.offby1.cc` redirects there (`httproute.yaml`).
No secrets and no volumes.

## Upgrades

Every push to `main` in 0xc0-homelab/offby1.cc publishes
`ghcr.io/0xc0-homelab/offby1.cc` as `sha-<7>` and `main`, and its
`container-image` job prints the image as `tag@digest` in its summary. An
upgrade is that string in `deployment.yaml`, in a PR here; Argo CD rolls it
out one pod at a time, never below two ready (`maxUnavailable: 0`).

The package on GHCR is public, so the cluster pulls with no secret.

## Headers

Traefik sets the security headers that are the same everywhere
(`security-headers` in `httproute.yaml`); the app sets its own
Content-Security-Policy, with a nonce per request.
