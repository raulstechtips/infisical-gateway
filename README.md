# Infisical Gateway

## Purpose

This wrapper uses the unchanged Infisical Gateway chart 1.2.0 from Cloudsmith.
The upstream chart appVersion is 1.1.0; its default CLI image is
`infisical/cli:0.43.84`. The wrapper adds one InfisicalStaticSecret for the
enrollment token. The pipeline publishes the chart to GHCR.

## Before first install

1. Make sure the Infisical operator supports `secrets.infisical.com/v1beta1`.
   The shared `InfisicalAuth` named `infisical-prod-universal-auth` in `default`
   must be ready. It is managed outside this chart.
2. Make sure `/infisical/gateway` in project `k3s-ntpo`, environment `prod`,
   contains `GATEWAY_ENROLLMENT_TOKEN`. Never put the token in Helm values.
3. Use `cluster-config/apps/dev/infisical-gateway.yaml` for this environment.
   Keep the release name and namespace as `infisical-gateway`.
4. For adoption, complete the steps in **Adoption notes** before the first sync.
   Set `targetRevision` in `argo/dev/infisical-gateway.yaml` to the first CI
   version after the new repository is merged. `PENDING_CI_VERSION` is not a
   version that Argo can download.
5. Sync manually. Check the StaticSecret status, gateway readiness and PVC
   binding. A fresh install needs a valid enrollment token. Adoption uses
   the existing persisted credentials; do not remove the volume to enroll again.

## Values reference

| Value | Default | Use |
| --- | --- | --- |
| `infisical.authRef.name`, `.namespace` | empty | Shared InfisicalAuth reference |
| `infisical.projectSlug`, `.environmentSlug` | empty | Source project and environment |
| `enrollmentSecret.enabled` | `false` | Enable the StaticSecret |
| `enrollmentSecret.secretName` | `infisical-gateway-enrollment-token` | Object and target Secret name; must match the upstream reference |
| `enrollmentSecret.secretPath` | `/infisical/gateway` | Source path |
| `enrollmentSecret.recursive` | `true` | Include child paths, as live |
| `enrollmentSecret.refreshInterval` | `60s` | Source refresh interval |
| `infisical-gateway` | object | Values passed unchanged to upstream |

Only upstream overrides are listed in `charts/values.yaml`. The default image,
5Gi capacity, ReadWriteOnce access mode, enabled persistence, empty existingClaim,
Service and security settings already match live. Resource overrides preserve
the live 10m CPU request and 256Mi memory limit.

The environment file holds the gateway name, domain, relay name, storage class
and Infisical connection. It enables token enrollment with those settings,
because upstream requires the gateway name and domain when enrollment is on.
The default render leaves enrollment off and is a validation baseline, not the
deployment configuration. No IngressRoute is needed for this gateway.

## Upgrades

Use `/opt/homebrew/bin/helm`. Change the dependency version only after checking
upstream changes, then run `helm dependency update charts` with isolated Helm
configuration. Keep the generated `Chart.lock`. Do not edit upstream templates.

Run the local CI checks:

```sh
export TMPDIR=/tmp/ulw-wrappers/infisical-gateway/
mkdir -p "$TMPDIR"
../.omo/wrapper-charts/tools/checks.sh "$PWD" "$TMPDIR"
```

Render with the environment values and compare with the live Application before
adoption or an upgrade. Never change storage class, claim name, capacity,
access modes or `existingClaim` as part of this adoption.

## Adoption notes

The parity target is the live Application, not the uncommitted old
`apps/infisical-gateway/values/prod.yaml`. That file was not changed.
Live discovery on October 30, 2026 showed six tracked resources and
Healthy/Synced status. The five upstream resources must stay SAME:
Deployment, Service, ServiceAccount, ClusterRoleBinding and PVC.

The intended secret migration has two differences:

- NEW `InfisicalStaticSecret/infisical-gateway-enrollment-token`: use the shared
  v1beta1 authentication reference. Preserve the live source path, recursive
  lookup, Opaque type and single `enrollment-token` key mapped from
  `GATEWAY_ENROLLMENT_TOKEN`. Refresh remains 60 seconds. The target policy
  changes from Owner to Orphan so deletion cannot delete the token Secret.
  Standard wrapper labels are added only to the new wrapper-owned object.
- PRUNE `InfisicalSecret/infisical-secret-infisical-gateway-environment`: the old
  controller must stop before the new controller manages the same Secret.

Before the first sync, the operator must orphan-delete the old resource:

```sh
kubectl delete infisicalsecret infisical-secret-infisical-gateway-environment \
  -n infisical-gateway --cascade=orphan
```

This is a future operator step. It was not run during local validation. Do not
delete the target Secret or the PVC. Then sync the new Application manually
and confirm the StaticSecret is ready.

The live Application has no sync options or ignoreDifferences to retain.
The new Application uses project `platform`, GHCR and manual sync, with
`Delete=false` to protect the Application. Upstream 1.2.0 has no `crds/` folder,
and the live Application tracks no CRDs; `skipCrds` is not needed.

The local checks passed with exit code 0: 3 test suites, 16 tests. Kubeconform
validated 5 resources and skipped the StaticSecret in the all-on render; the
default render validated 4 resources. Both had 0 invalid resources and 0 errors.
The development render parity result was:

```text
SUMMARY SAME=5 CHANGED=0 NEW=1 PRUNE=1 HOOK=0 UNKNOWN-KIND=0
```

All five upstream resources, including the PVC, are SAME. The only differences
are the secret migration described above. Written YAML, the dependency lock,
JSON schema and rendered manifests passed yq parsing. Helm templates were
parsed after rendering.

Evidence is in `../.omo/evidence/wrappers/infisical-gateway/`.
