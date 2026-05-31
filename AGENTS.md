# AGENTS.md — ingress-caddy (fork)

This repo is a fork of `brdelphus/ingress-caddy`, a Helm chart that turns Caddy into a
Kubernetes ingress controller. This fork adds features required by the homelab stack.
The chart is consumed by the `caddy-edge` HelmRelease in `k3s-cluster/`.

The companion `caddy-edge/` repo builds the custom Caddy image that this chart deploys.
Changes to chart features and image builds are often paired — see cross-repo note below.

---

## What This Fork Adds

| Feature | Status |
|---------|--------|
| GeoIP allowlist mode (`allow_countries AT`) | Implemented |
| LAN/internal CIDR bypass for GeoIP | Implemented |
| Cloudflare trusted proxy auto-refresh (`caddy-cloudflare-ip`) in Caddyfile template | Implemented |
| Named Caddyfile snippets available in `extraSiteBlocks` | Implemented |
| Per-Ingress annotation overrides for GeoIP and CrowdSec behavior | In progress |

Named snippets available to `extraSiteBlocks` consumers:

| Snippet | What it does |
|---------|-------------|
| `(internal_ranges)` | Matches IPs in `realIP.internalRanges` |
| `(external_ranges)` | Inverse of `(internal_ranges)` |
| `(security)` | Coraza WAF + `@geoblock` (AT-only, LAN bypassed) + security headers |

---

## Repo Structure

```
helm/ingress-caddy/
  Chart.yaml          Chart metadata and version
  values.yaml         All feature toggles and defaults
  templates/          Caddyfile generation + K8s resource templates
examples/             Example values files for common deployment patterns
docker/               Dockerfile for the upstream image (not the caddy-edge fork)
```

Changes to chart behavior live in `templates/` and `values.yaml`. The Caddyfile is
generated from templates — do not hand-edit output; edit the template.

---

## Upstream Reference

Upstream: `brdelphus/ingress-caddy`. When tracking upstream changes:
1. Review upstream `CHANGELOG.md` and diff `templates/` before merging.
2. The `caddy-k8s` and `caddy-kubernetes-storage` module pins in the upstream image
   must stay in sync with the commit hashes used in `caddy-edge/Dockerfile`.
3. Never blindly merge upstream — the GeoIP allowlist logic conflicts with upstream's
   deny-only mode.

---

## Production Caddyfile Patterns to Preserve

These patterns from the old hand-crafted Caddyfile must remain expressible via chart values
or `extraSiteBlocks`:

- `allow_countries AT` with LAN bypass (`import security` + `import external_ranges`)
- Selective CrowdSec: only geofiltered (external) requests hit the bouncer
- Per-site Authentik `forward_auth` (external clients only, LAN bypasses)
- Admin path blocking from external IPs
- Per-site `Content-Security-Policy` frame-ancestors override via annotation

If a chart change breaks any of these patterns, it is a regression regardless of whether
tests pass.

---

## Versioning and Release

- `VERSION` file holds the current chart version.
- `CHANGELOG.md` tracks changes.
- Bump `version` in `Chart.yaml` and `VERSION` for every release.
- The consuming HelmRelease in `k3s-cluster/infrastructure/services/caddy-edge/helmrelease.yaml`
  pins an exact chart version — a chart release is only live when that pin is updated and
  Flux reconciles.

Cross-repo order for a chart feature change:
1. Implement and version-bump in this repo, commit + push.
2. Update the chart version pin in `k3s-cluster/`.
3. If the feature requires a new Caddy module, update `caddy-edge/Dockerfile` too (before
   or alongside step 2).

---

## What Not to Change

- Do not modify upstream plugin list or xcaddy build args here — that lives in `caddy-edge/Dockerfile`.
- Do not hardcode internal hostnames or IPs into chart templates — those belong in values
  overrides in `k3s-cluster/`.
- Do not add secrets or credentials to any file in this repo.
