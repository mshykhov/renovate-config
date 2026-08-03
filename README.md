# renovate-config

Shared update policy for my repositories. Two layers, wired in independently:

| Layer | Decides | For |
|---|---|---|
| `renovate-config` | update type | everyone |
| `renovate-config:k8s-platform` | classification of the usual cluster components | anyone running Kubernetes |

The second layer is separate on purpose: it is an opinion about specific software rather than
about update types, and a repository without a cluster has no use for it.

## Wiring it into a repository

1. **Install Renovate on the repository** (if it is not there yet) - the
   [Mend Renovate](https://github.com/apps/renovate) app, with the repository selected in its
   settings. Nothing needs to be installed on this repository: it is public, and Renovate reads
   presets from public repositories without being installed on them.

2. **Put `renovate.json` in the root** of your repository:

   ```json
   {
     "$schema": "https://docs.renovatebot.com/renovate-schema.json",
     "extends": [
       "github>mshykhov/renovate-config",
       "github>mshykhov/renovate-config:k8s-platform"
     ]
   }
   ```

   Drop the second line if there is no cluster.

3. **Validate before pushing** - the validator resolves remote presets too, so a typo in the
   `extends` line surfaces immediately:

   ```bash
   npx --yes --package renovate -- renovate-config-validator
   ```

   Run it with no arguments, from the repository root: given a path, it validates against the
   global-config schema instead and passes files it should reject.

4. **Confirm the preset landed** - after its first run Renovate opens a "Dependency Dashboard"
   issue. An unresolved preset shows up there as its own error block.

Local rules go on top of the presets - see "What the presets do not decide" below.

A live example with every local exception:
[smhomelab-infrastructure/renovate.json](https://github.com/mshykhov/smhomelab-infrastructure/blob/master/renovate.json).

## What the base preset decides

Only what does not depend on the project - the update type:

| Type | Behaviour |
|---|---|
| `digest` / `pin` | automerged |
| `patch` | automerged, 14 day hold |
| `minor` | PR, merged by hand |
| `major` | PR only after a Dependency Dashboard tick, never automerged |

Automerge runs at night (01:00-05:00 UTC): a bad update should not land during working hours,
least of all where merging means deploying.

The hold before an automerge is longer than the global one (14 days against 7) deliberately. An
automerged release is one nobody read before it shipped, and two weeks is roughly what it takes
for a compromised package to be pulled from its registry.

## What the k8s-platform layer decides

It classifies the usual cluster components by one question - **what does a rollback cost**:

| Group | Behaviour | Why |
|---|---|---|
| external-dns, cert-manager, reloader, tailscale-operator, blackbox-exporter, image-updater | minor + patch automerged | own no data, worst case is a revert and a resync |
| longhorn, vault, external-secrets | by hand | a bad upgrade takes the data with it, and a revert does not bring it back |
| cloudnative-pg, plugin-barman-cloud | by hand, one group | operator, cluster chart and backup plugin move together or not at all |
| mariadb, mysql, postgres, redis, valkey, mongo | by hand | a database upgrades its own on-disk state on first start |
| traefik, ingress-nginx, authentik, keycloak | by hand | breaking one locks everything behind it out, including the GitOps controller used to fix it |
| argo-cd, flux2 | by hand | a broken upgrade cannot be fixed by the thing that is broken |
| `*-operator`, CRDs | by hand | reconciliation state, and schemas that a downgrade does not prune |

These names live here rather than in every repository because what they are does not change
between clusters: Longhorn owns volumes wherever it runs, and external-dns owns nothing wherever
it runs.

## What the presets do not decide

Anything whose meaning is **local**: an image built in the repository itself, a version pinned to
match another component, a node upgraded by hand. Such rules must not be shared - in someone
else's repository they are either useless or harmful.

```json
{
  "extends": [
    "github>mshykhov/renovate-config",
    "github>mshykhov/renovate-config:k8s-platform"
  ],
  "packageRules": [
    {
      "description": "Must match the helm bundled in argocd-repo-server, not the newest helm.",
      "matchManagers": ["github-actions"],
      "matchDepNames": ["helm"],
      "enabled": false
    }
  ]
}
```

Repository rules come after the extended presets and therefore override them.

## Prerequisite

Renovate only automerges on green checks. In a repository without CI, automerge proves nothing -
checks first, then this preset.
