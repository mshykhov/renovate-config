# Renovate presets

Shared dependency-update policies. `default.json` controls update types; `k8s-platform.json` adds rules for Kubernetes components.

## Use

Enable [Renovate](https://github.com/apps/renovate) on your repository and add `renovate.json`:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>mshykhov/renovate-config",
    "github>mshykhov/renovate-config:k8s-platform"
  ]
}
```

Omit the second preset for projects without Kubernetes. Repository settings and later package rules can override these defaults. Check the Dependency Dashboard after Renovate's first run for configuration errors.

## Base policy

| Update | Default handling |
|---|---|
| Digest / pin | Automerge when eligible |
| Patch | Automerge after a 14-day release-age delay |
| Minor | Manual merge |
| Major | Dashboard approval before PR creation; manual merge |
| Lock-file maintenance | Weekly, before 05:00 Monday UTC; manual merge |

The global release-age delay is 7 days. Automerge is scheduled for 01:00-05:00 UTC and performed by Renovate rather than platform automerge. Required checks and branch protection should be configured in each consuming repository. Release age is a delay, not a guarantee that an update is safe.

`rangeStrategy: "update-lockfile"` updates supported lock files for in-range releases. For applications that need exact deployed versions, use exact dependency pins and inspect the resulting lock-file diff.

## Kubernetes policy

- Selected components such as external-dns, cert-manager, reloader, and image-updater allow minor and patch automerge.
- Storage, databases, identity, ingress, GitOps controllers, and operator/CRD changes require manual merge.
- CloudNativePG and its Barman plugin are grouped; Longhorn minor upgrades are kept separate.

See [k8s-platform.json](k8s-platform.json) for the exact package matchers. Review them against your cluster before adopting the preset.

## Validate

Use a Node.js version supported by the current Renovate release. From a consuming repository, validate `renovate.json`:

```bash
npx --yes --package renovate@latest -- renovate-config-validator
```

From this repository, validate both preset files as repository configuration:

```bash
npx --yes --package renovate@latest -- renovate-config-validator --no-global default.json k8s-platform.json
```

The [`--no-global` option](https://docs.renovatebot.com/config-validation/) matters when passing custom filenames.

[MIT License](LICENSE)
