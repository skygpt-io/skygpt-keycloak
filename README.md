# skygpt-keycloak

D2-architecture **producer repo** for the Keycloak Operator on the
SkyGPT platform. Holds vendored install manifests and publishes them
as a cosign-signed OCI artifact that `skygpt-infra` consumes.

> **Producer-only**. This repo does NOT deploy Keycloak anywhere. It
> ships a versioned manifest bundle. The actual deployment lives in
> [`skygpt-infra/components/keycloak/`](https://github.com/skygpt-io/skygpt-infra/tree/main/components/keycloak)
> as a thin consumer that pulls the signed artifact from this repo.

## What this repo ships

- **`kubernetes/`** — vendored manifests from
  [`keycloak-k8s-resources`](https://github.com/keycloak/keycloak-k8s-resources)
  at tag **26.6.2**:
  - `kubernetes.yml` — operator install (Deployment + RBAC + Service +
    SA). Carries `metadata.labels."skygpt.io/role": platform-utility`
    on the operator Deployment for REQ-D2-004 compliance downstream.
  - `keycloaks.k8s.keycloak.org-v1.yml` — `Keycloak` CRD
  - `keycloakrealmimports.k8s.keycloak.org-v1.yml` — `KeycloakRealmImport` CRD
  - `kustomization.yaml` — applies the upstream-mirror image override
    (`quay.io/keycloak/keycloak-operator` → mirrored path)

## Published artifact

On tag `v*` push, `.github/workflows/release-oci.yaml`:
1. Re-validates the bundle (`kustomize build` + kubeconform)
2. Pushes
   `us-east4-docker.pkg.dev/skygpt-platform/skygpt-registry/skygpt-keycloak:<tag>`
3. Cosign keyless-signs the digest

Consumers verify with the OIDC identity:

```
^https://github\.com/skygpt-io/skygpt-keycloak/\.github/workflows/release-oci\.yaml@refs/tags/v.*$
```

Same WIF identity model as
[`skygpt-policy`](https://github.com/skygpt-io/skygpt-policy) — see
`platform-identity` in `skygpt-foundation` for the publisher provider
that mints the GCP auth tokens used by this repo's workflows.

## Updating to a newer Keycloak Operator release

Refresh procedure (matches the pattern in skygpt-infra's keycloak
component README; this is the canonical place now):

```bash
cd kubernetes
VERSION=<new tag, e.g. 26.7.0>
for f in kubernetes.yml keycloaks.k8s.keycloak.org-v1.yml keycloakrealmimports.k8s.keycloak.org-v1.yml; do
  curl -fsSL "https://raw.githubusercontent.com/keycloak/keycloak-k8s-resources/${VERSION}/kubernetes/${f}" -o "$f"
done
```

**Critical**: the d2-compliance check in `skygpt-infra` inspects raw
YAML, not kustomize render output, so the `platform-utility` label
must be re-applied on the operator Deployment in `kubernetes.yml` after
each refresh:

```yaml
# inside the Deployment in kubernetes.yml
metadata:
  labels:
    ...
    skygpt.io/role: platform-utility
```

`./scripts/validate.sh` in skygpt-infra (downstream) enforces this; it
will fail loudly if the label is missing on consumption.

Then update this README's vendored-version reference, open a PR. The
release-please bot will propose `v<next>` once merged; merging the
release PR triggers a new signed artifact.

## Consumer wrapper

See [`skygpt-infra/components/keycloak/controllers/base/skygpt-keycloak.yaml`](https://github.com/skygpt-io/skygpt-infra/tree/main/components/keycloak/controllers/base)
for the consumer wrapper — it pins this repo's tag via an
`OCIRepository` + cosign verify, then references it from a Flux
`Kustomization`. Per-env deltas (Keycloak CR config, the CNPG Postgres
backing, the realm import) live in the skygpt-infra component
alongside the wrapper.

## License

[Apache 2.0](./LICENSE) — same as the vendored upstream and the rest
of the SkyGPT platform repos.
