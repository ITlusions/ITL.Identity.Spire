# ITL.Identity.Spire

Self-hosted [SPIRE](https://spiffe.io/docs/latest/spire-about/) (the SPIFFE Runtime Environment) for the ITL Talos cluster ([`ITlusions/ITL.K8s`](https://github.com/ITlusions/ITL.K8s)) - issues short-lived, cryptographic workload identities (X.509-SVID / JWT-SVID) per pod via Kubernetes node attestation, so services can authenticate to each other (mTLS) without hand-distributed certs or shared secrets.

**Not a replacement for Keycloak** - Keycloak (`ITL.Keycloak.Tenants`, and the OIDC clients used by `ITL.Headscale`, `ITL.AuthGateway`, etc.) stays the identity provider for humans and user-facing login flows. SPIRE is the orthogonal workload-to-workload layer: *this specific pod, on this specific node, is who it claims to be* - independent of any human session.

**Status: deployed and live** on the `itl` cluster (namespace `itl-spire`), as of 2026-10-30 -
113+ registration entries, cluster-wide auto-registration active, two real consumers
(`ITL.Openproject.fork`, `examples/svid-toggle-demo` in this repo) and one reusable
Kyverno-based sidecar-injection policy (`ITL.Kyverno`'s `spiffe-helper-inject`) built on top of
it. See [`docs/index.md`](docs/index.md) for operational docs (the SPIRE CLI, mainly).

## This is a dependency wrapper, not hand-rolled manifests

Unlike the other `ITL.*` app charts (`ITL.Headscale`, `ITL.Wireguard`, ...), which write their own Deployment/StatefulSet/RBAC manifests against upstream container images, this chart tracks the **upstream [`spiffe/helm-charts-hardened`](https://github.com/spiffe/helm-charts-hardened) `spire` chart as a real Helm dependency** (see `Chart.yaml`). `values.yaml` here only overrides the handful of keys that are actually cluster-specific (trust domain, cluster name, storage class) - everything else is upstream's own hardened default. Chosen because SPIRE's deployment surface (server, agent, the CSI driver, the controller-manager, all their RBAC) is large and upstream actively maintains a "hardened" chart for exactly this - re-implementing it by hand would just drift from upstream security fixes over time, unlike e.g. Headscale where this org's own thin wrapper is simple enough to maintain directly.

**The CRDs chart (`spire-crds`) is intentionally NOT a `Chart.yaml` dependency** - upstream's own docs install it as a separate Helm release before the main chart, specifically because CRD lifecycle (install/upgrade/never-auto-delete) doesn't fit normal subchart semantics. See "Deploy" below.

## Architecture

- **spire-server** (StatefulSet, SQLite datastore on the `ha-gen-lh` Longhorn storage class) - holds the trust domain's root CA and the registration-entry database.
- **spire-agent** (DaemonSet, one per node) - attests to the server via Kubernetes PSAT (projected service account token), then exposes the local Workload API over a Unix socket.
- **spire-controller-manager** (bundled under `spire-server` upstream, enabled here) - lets registration entries be managed as CRDs/manifests instead of one-off `spire-server entry create` CLI calls, matching this org's GitOps pattern. Wired up via one cluster-wide `default` `ClusterSPIFFEID` (empty `namespaceSelector`, matching every workload) - see "Registration" below for what that actually does and doesn't cover.

```
workload pod --(unix socket)--> spire-agent (DaemonSet, per node) --(gRPC)--> spire-server (StatefulSet)
                                       ^
                                 k8s_psat node attestation
                                 k8s workload attestation (pod labels/SA via API server)
```

## Trust domain

`itlusions.local` - deliberately not a resolvable public hostname; it's the permanent root of
every SVID this cluster issues, confirmed live in every registration entry (`spiffe://itlusions.local/ns/...`).
**Changing it now means re-issuing every workload identity from scratch** - treat it as fixed. If
a second ITL cluster ever runs its own SPIRE server, decide then whether it shares this trust
domain or gets its own; this chart assumes single-cluster scope and has no federation configured.

## Why SQLite

Same reasoning as `ITL.Headscale`: fewer moving parts than standing up Postgres for a single-server control plane, and it matches this org's existing SQLite-on-PVC pattern (`ITL.Headscale`, `ITL.Wireguard`). This is upstream's own default persistence mode, not something this wrapper added - just pointed at this cluster's `ha-gen-lh` storage class. SPIRE server HA requires Postgres + multiple replicas - not configured here, deliberately deferred; would be an upstream value override (`global.spire.persistence`), not a change to this wrapper's structure.

## No upstream CA

Upstream's default self-signs its own root for the trust domain - not overridden here. Fine for getting workload identity running; revisit if this trust domain's root key should ever live outside the cluster (Vault PKI, AWS Private CA, etc. as an UpstreamAuthority plugin) - that would be a `values.yaml` addition under `spire.spire-server`, not a manifest change, since upstream owns the manifests.

## Registration: every workload gets an identity, nothing auto-injects it

`values.yaml`'s `registrationEntries` has one entry by default (`default`, `templates/clusterspiffeid.yaml`'s
render of it) - an empty `namespaceSelector`, matching every workload cluster-wide. **This only
controls who gets a registration entry (an identity *issued*)** - it does not put a `spiffe-helper`
sidecar, a CSI volume, or anything else into any pod. Turning an issued identity into something a
container can actually read is still each consumer's own concern - either hand-rolled
(`ITL.Openproject.fork`'s own chart), or via `ITL.Kyverno`'s reusable `spiffe-helper-inject`
`ClusterPolicy` (one pod annotation, `spiffe.itlusions.local/inject: "true"` - see
`examples/svid-toggle-demo` for a live example of both approaches in one chart).

The default entry's `dnsNameTemplates` sets each leaf SVID's CN to
`{namespace}-{app.kubernetes.io/name, or "workload" if unset}` - SPIRE always overwrites the CN
with `DNSNames[0]` whenever any DNS name is set (confirmed in SPIRE's own
`pkg/server/credtemplate/builder.go`), which is what makes CNPG's `clientcert=verify-full`
(CN-matching, not SPIFFE-SAN-aware) usable at all. Namespace-prefixed specifically because the
bare label collides across unrelated workloads - four separate CNPG Postgres clusters in
different namespaces all happened to carry `app.kubernetes.io/name=postgresql`. Confirmed live
the hard way that a template rendering to an empty string (rather than failing to render) gets the
*whole* ClusterSPIFFEID update rejected by the server, not just that one entry - 33 real workloads
briefly lost their identity entirely before the current if/else fallback was added.

The CLI fallback for a one-off, hand-registered entry (what `registrationEntries` exists to avoid
needing routinely) - see [`docs/1-operations/1.1-spire-cli.md`](docs/1-operations/1.1-spire-cli.md)
for this and every other `spire-server`/`spire-agent` command actually confirmed live against this
cluster.

## Open items, not yet resolved

- **The throwaway PoC entry (`spire-poc-test`) is still in `values.yaml`'s `registrationEntries`**,
  flagged for removal once a real workload existed to prove the chain end-to-end instead - which
  `examples/svid-toggle-demo` and `ITL.Openproject.fork` have both now done. Safe to delete; not
  done yet because nothing has needed to touch `values.yaml` since.
- **SPIRE server HA** (Postgres + multiple replicas instead of the current single-server SQLite
  setup) - deliberately deferred, see "Why SQLite" below; would be an upstream value override, not
  a change to this wrapper's structure.
- **No UpstreamAuthority plugin** - this trust domain's root key is self-signed and lives only in
  this cluster (see "No upstream CA" below); revisit if it should ever live in Vault PKI/AWS
  Private CA instead.
- **Federation with a second trust domain** is unexplored - only relevant if a second ITL cluster
  ever runs its own SPIRE server (see "Trust domain" below).

## Deploy

Two-step, per upstream's own documented process - the CRDs chart is a separate Helm release, not a dependency of this one (see "This is a dependency wrapper" above):

```bash
# 1. CRDs first, as their own release, their own lifecycle
helm upgrade --install spire-crds spire-crds \
  --repo https://spiffe.github.io/helm-charts-hardened/ \
  -n itl-spire --create-namespace

# 2. This wrapper (pulls in the upstream "spire" chart as its dependency)
cd itl.identity.spire
helm dependency update
helm upgrade --install spire . -n itl-spire
```

## Verify

```bash
kubectl get pods -n itl-spire
# spire-server-0, one spire-agent pod per node, spire-controller-manager, spire-spiffe-oidc-discovery-provider
```

See [`docs/1-operations/1.1-spire-cli.md`](docs/1-operations/1.1-spire-cli.md) for `healthcheck`,
`entry show`/`entry create`/`entry delete`, `agent list`, `bundle show`, and a practical
one-liner for working with `entry show`'s plain-text output at this cluster's actual scale
(100+ entries) - every command there confirmed live, not transcribed from upstream docs.

## See also

- [`examples/svid-toggle-demo`](examples/svid-toggle-demo/) - a three-tier app demonstrating
  X.509-SVID (via `ITL.Kyverno`'s annotation), JWT-SVID (hand-rolled), and where a classic
  CN-matching database connection needs `ITL.CertManager.CA` instead - SPIFFE SVIDs were never
  going to satisfy CNPG's `clientCASecret` as-is.
- [`docs/index.md`](docs/index.md) - the full docs index.
