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
       "local>mshykhov/renovate-config",
       "local>mshykhov/renovate-config:k8s-platform"
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

### Where there is a lock file

The table above only works if Renovate has an update to classify, and in a repository whose
manifest is written in open ranges - `httpx >=0.27`, `fastapi >=0.115`, `^1.2.0` - it has none.
Every new release already satisfies the range, so there is nothing in the manifest to edit, and
the version that actually ships is the one written in `uv.lock` / `package-lock.json` /
`Cargo.lock`. Left alone, that file never moves and the repository quietly stops receiving
updates at all, security fixes included, while the dashboard looks healthy.

So the preset sets `rangeStrategy: "update-lockfile"`: an in-range release updates the lock and
leaves the manifest untouched, an out-of-range one edits the manifest as before. Either way it is
a normal update with a real `patch`/`minor` type, so everything in the table applies to it -
patches automerge after their 14 days, minors wait for a human.

`lockFileMaintenance` runs weekly on top of that, for the transitive dependencies that appear in
no manifest and that nothing else would ever propose. It is **not** automerged, and that is not an
oversight: it deletes the lock and regenerates it at whatever is newest, which is the one update
path the 14-day hold does not cover.

The trade-off is deliberate: the manifest stops recording which version is in use. The lock is
what the build reads, so that is where the truth is kept.

### Where the manifest is a deployed application, pin instead

The above is the right shape for a library, whose ranges exist so that consumers can resolve
them. For an application it is worth pinning exactly - `fastapi==0.139.0`, not `fastapi>=0.115` -
and letting the manifest stay in charge.

Beyond reproducibility there is a concrete failure behind this. Measured with uv on 2026-08-03:
Renovate picks the version that has cleared its hold and asks the package manager for it, but
`uv lock --upgrade-package fastapi` resolves to the newest release that satisfies the range and
ignores that choice. Renovate had selected the held `0.140.2`; the lock came out at `0.141.1`,
five days old. The hold this preset is built around was bypassed, and every such PR failed its
own artifact check - visible, but as a broken PR rather than as a policy.

With an exact pin the requested version is the only one that satisfies the manifest, so the
package manager has nothing to choose and the hold means what it says.

## What the k8s-platform layer decides

It classifies the usual cluster components by one question - **what does a rollback cost**:

| Group | Behaviour | Why |
|---|---|---|
| external-dns, cert-manager, reloader, tailscale-operator, blackbox-exporter, image-updater | minor + patch automerged | own no data, worst case is a revert and a resync |
| longhorn, vault, external-secrets | by hand | a bad upgrade takes the data with it, and a revert does not bring it back |
| longhorn | additionally, one PR per minor | it supports a single minor step at a time and enforces it - the pods refuse to start on a skipped path rather than warn |
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
