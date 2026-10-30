# ITL.Identity.Spire documentation index

See the top-level [README](../README.md) for architecture, deploy, and open items. Documents below
are grouped into numbered folders, matching `ITL.Headscale/docs/`'s pattern - each file named
`x.y-title.md`, where `x` is the folder number.

## 1. Operations

- [1.1 Using the SPIRE CLI](1-operations/1.1-spire-cli.md): `spire-server`/`spire-agent` commands
  confirmed live against this cluster - registration entries, the trust bundle, node attestation,
  healthchecks, and a practical PowerShell one-liner for working with `entry show`'s plain-text
  output at this cluster's actual scale (100+ entries).
- [1.2 Tornjak: a web UI for SPIRE](1-operations/1.2-tornjak-ui.md): enabling Tornjak (already a
  conditional dependency of the upstream `spire` chart), exposing it via a single basicAuth-gated
  Ingress with the backend never public on its own, and the real bugs hit along the way - the
  `apiServerURL` auto-detection trap, Traefik's one-key-per-Secret `basicAuth` requirement, and
  why "Tornjak clusters" are a purely cosmetic UI concept separate from SPIRE's own agents.
- [1.3 OIDC discovery: exposing the trust bundle as JWKS](1-operations/1.3-oidc-discovery.md):
  exposing the SPIFFE OIDC discovery provider (already enabled by upstream's own chart default)
  publicly at `https://trust.itlusions.com`, deliberately with no auth gate - why TLS is disabled
  at the provider itself in favor of edge termination, why `ingress.tlsSecret` has to be set
  explicitly for cert-manager to pick up the host, and the live discovery document/JWKS response
  confirming this trust domain's actual signing keys. Groundwork for Keycloak JWT-SVID validation
  in a later phase - Keycloak and the demo chart are untouched here.
- [1.4 Keycloak token exchange: trading a JWT-SVID for a Keycloak access token](1-operations/1.4-keycloak-token-exchange.md):
  registering SPIRE's trust bundle as an `itlusions`-realm identity provider, the issuer-string
  mismatch this uncovered between `trust.itlusions.com` and SPIRE's actual `iss` claim, why
  Keycloak 26.8's RFC 7523 JWT Authorization Grant (not the `token-exchange` grant type) turned
  out to be the real mechanism here, and the one credential-handling step still left for a human
  to finish the live end-to-end verification.
