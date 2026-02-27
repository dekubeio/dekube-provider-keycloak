# dekube-provider-keycloak

![vibe coded](https://img.shields.io/badge/vibe-coded-ff69b4)
![python 3](https://img.shields.io/badge/python-3-3776AB)
![heresy: 3/10](https://img.shields.io/badge/heresy-3%2F10-green)
![stdlib only](https://img.shields.io/badge/dependencies-stdlib%20only-brightgreen)
![public domain](https://img.shields.io/badge/license-public%20domain-brightgreen)

Keycloak CRD converter for [dekube](https://dekube.io).

## Handled kinds

- `Keycloak` -- converts the Keycloak CR into a compose service
- `KeycloakRealmImport` -- converts realm import CRs into JSON files for auto-import on startup

## What it does

Replaces the Keycloak Operator's reconciliation logic with a direct compose service. The K8s operator Deployment itself is ignored (K8s-only); this provider generates the Keycloak server container directly from the CR spec.

**Keycloak CR:**
- Maps `spec.db` to `KC_DB_*` env vars (host, port, credentials from referenced Secrets)
- Maps `spec.http`, `spec.hostname`, `spec.proxy` to corresponding `KC_*` env vars
- Forces `KC_CACHE=local` (compose = single instance, no Infinispan clustering)
- Enables health + metrics (`KC_HEALTH_ENABLED`, `KC_METRICS_ENABLED`) and management port (default 9000)
- Mounts TLS secrets for HTTPS (`spec.http.tlsSecret`) as PEM files
- Processes `spec.unsupported.podTemplate.spec` for additional volume mounts (ConfigMaps and Secrets, e.g. CA trust bundles)
- Resolves `spec.features`, `spec.additionalOptions`, and pod-level env vars
- Generates bootstrap admin credentials if `spec.bootstrapAdmin` is absent (written to `secrets/<name>-initial-admin/`, reused across runs)
- Registers namespace and K8s Service alias (`<name>-service`) in `ctx.services_by_selector` and `ctx.alias_map` for network alias generation (FQDN resolution in compose DNS)

**KeycloakRealmImport CR:**
- Writes realm definitions as JSON files under `configmaps/<keycloak-name>-realms/`
- Resolves `${PLACEHOLDER}` values from referenced K8s Secrets
- Applies configured string replacements in realm data
- Skips master realm import (preserves Keycloak defaults)
- Adds `--import-realm` to the Keycloak start command

## Priority

`500` -- runs after cert-manager/trust-manager if present (uses their output when available).

## Dependencies

None. Uses stdlib only (`base64`, `json`, `os`, `secrets`, `string`, `sys`). Imports `ConvertResult` and `_apply_replacements` from `dekube`.

## Usage

Via dekube-manager (recommended):

```bash
python3 dekube-manager.py keycloak
```

Manual (pass the extension directory directly):

```bash
python3 helmfile2compose.py --extensions-dir ./dekube-provider-keycloak --helmfile-dir ~/my-platform -e local --output-dir .
```

## Code quality

*Last updated: 2026-02-23*

| Metric | Value |
|--------|-------|
| Pylint | 9.97/10 |
| Pyflakes | clean |
| Radon MI | 32.23 (A) |
| Radon avg CC | 4.6 (A) |

Worst CC: `_build_pod_template_volumes` (12, C), `_build_options_env` (11, C).

The `E0401: Unable to import 'dekube'` is expected — extensions import from dekube-engine at runtime, not at lint time.

## License

Public domain.
