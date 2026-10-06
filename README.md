# renovate

Shared [Renovate](https://docs.renovatebot.com/) config for my repositories.

Use it from a repo's `renovate.json`:

```json
{
    "$schema": "https://docs.renovatebot.com/renovate-schema.json",
    "extends": ["github>ansasi/renovate-config"]
}
```

## Presets

| File | What it does |
|---|---|
| `default.json` | Entry point, extends everything below |
| `autoMerge.json5` | Automerges minor, patch and digest updates after 3 days (branch automerge, no PR) |
| `labels.json5` | Adds a `type/<update type>` label |
| `semanticCommits.json5` | Conventional commit messages and scopes (`container`, `helm`, `github-action`…) |
| `databases.json5` | Stricter policy for database images, see below |
| `calver.json5` | No automerge for date versions (`2026.02.07`), see below |
| `kubernetes.json5` | File patterns for the `kubernetes`, `argocd` and `flux` managers |

## Databases

| Update | Behaviour |
|---|---|
| patch | Automerged after 7 days |
| minor, PostgreSQL | Automerged after 7 days |
| minor, other databases | PR, merged by hand |
| major | No PR until approved from the Dependency Dashboard |

All database updates get the `database` label.

### Why

- **Rollback is the problem.** When an app update breaks, reverting the PR fixes it.
  A database often rewrites its data on disk when it starts on a new version. Reverting
  the image tag then leaves an old version that can't read the data, and only a backup
  restore gets you back. With automerge on a repo that deploys from `main`, that could
  happen with nobody watching.
- **A "minor" bump is usually a new release series.** For example:
    - MySQL 8.0 → 8.4 turns off `mysql_native_password` by default, so some apps can't log in.
    - MariaDB 10.6 → 10.11 and 11.4 → 11.8 upgrade the data files, with no way back.
    - Elasticsearch: once a newer version opens an index, older versions can't read it.
- **PostgreSQL is the exception.** Its X.Y → X.Z releases are bug fixes with the same
  on-disk format, and the Postgres team recommends always running the latest one.
- **Majors need a planned migration** (dump/restore, `pg_upgrade`…). Automatic PRs for
  them are only noise, so they wait in the Dependency Dashboard until you tick them.
- **7 days instead of 3:** in November 2024, PostgreSQL 17.1 and 16.5 broke
  extensions such as TimescaleDB, and a fix came out about a week later.

### Where to configure database updates

- **Here:** the general policy, because the risk is the same in every repo. Copying
  rules into each repo means they drift apart, and new repos start out unprotected.
- **In the app's repo:** limits that belong to one app only. For example:

```json5
{
    packageRules: [
        // Nextcloud only supports up to a certain MariaDB version
        { matchPackageNames: ["mariadb"], allowedVersions: "<=11.4" },
        // Immich ships its own Postgres image; follow the Immich release notes instead
        { matchPackageNames: ["ghcr.io/immich-app/postgres"], enabled: false },
        // This Redis is a disposable cache, so minors can automerge
        { matchPackageNames: ["redis"], matchUpdateTypes: ["minor"], automerge: true },
    ],
}
```

Don't use `ignoreDeps` to silence a database: you would also stop getting its security
patches.

Pin full versions in compose files (`postgres:16.4`, not `postgres:16`). With a short
tag, Renovate can only suggest majors, and the updates in between happen silently on
`docker compose pull`.

## Date versions (CalVer)

Dependencies whose current version starts with a date (`2026.02.07`, `2026-02-07`,
`v2026.02.07`, `2025.10.1`…) are never automerged. They get a PR with the `calver` label,
merged by hand.

Automerge trusts the update type: with SemVer (`1.3.2`) a minor or patch promises no
breaking change. A date only says when the release came out, yet Renovate still calls
`2026.02.07 → 2026.03.01` a "minor" update and would automerge it.

To get automerge back in one repo, add a rule to its `renovate.json`. Repo rules run after
the presets, so they win:

```json5
{
    packageRules: [
        // Only this package
        { matchPackageNames: ["ghcr.io/home-assistant/home-assistant"], automerge: true },
        // Or every date version in this repo
        { matchCurrentVersion: "/^v?20\\d{2}[.-]/", automerge: true },
    ],
}
```

Or drop the preset completely:

```json
{
    "ignorePresets": ["github>ansasi/renovate-config:calver.json5"]
}
```

## Kubernetes

The `kubernetes` and `argocd` managers scan no files by default, and `flux` only reads
its own `gotk-components.yaml`. `kubernetes.json5` points them at `k8s/`,
`kubernetes/`, `cluster(s)/`, `manifests/`, `apps/` and `infrastructure/`. Each manager
only reads files that look like its own manifests, so other YAML in those folders is
ignored. A repo with another layout can add its own patterns, which are merged with these:

```json
{
    "kubernetes": { "managerFilePatterns": ["/deploy/.+\\.ya?ml$/"] }
}
```
