# renovate

Shared [Renovate](https://docs.renovatebot.com/) config for my repositories.

Use it from a repo's `renovate.json`:

```json
{
    "$schema": "https://docs.renovatebot.com/renovate-schema.json",
    "extends": ["github>ansasi/renovate-config"]
}
```

`local>ansasi/renovate-config` works too. The presets reference each other with relative
paths (`./autoMerge.json5`), so they are always read from the same place as `default.json`.

## Requirements

**Renovate 44 or newer** (tested with 44.145.1). Older versions stop with a configuration
error in every repo that uses these presets: Renovate 39, for example, doesn't know relative
preset paths, `managerFilePatterns`, `minimumReleaseAgeBehaviour` or `abandonments:recommended`.

The [Validate](.github/workflows/validate.yaml) workflow checks every preset on each push and
PR. Keep its `RENOVATE_VERSION` on the same version as the self-hosted runner, so a preset that
passes here also loads there.

## Presets

| File | What it does |
|---|---|
| `default.json` | Entry point, extends everything below, see [Other settings](#other-settings) |
| `releaseAge.json5` | Every update waits until the new version is 7 days old, see [Release age](#release-age-7-days) |
| `autoMerge.json5` | Automerges minor, patch and digest updates, except 0.x minors, see [Automerge](#automerge) |
| `labels.json5` | Adds a `type/<update type>` label (`type/major` for 0.x minors) |
| `semanticCommits.json5` | Conventional commit messages and scopes (`container`, `helm`, `github-action`…) |
| `databases.json5` | Stricter policy for database images, see below |
| `calver.json5` | No automerge for date versions (`2026.02.07`), see below |
| `supplyChain.json5` | npm, PyPI, crates.io and Go: the 7-day wait also for indirect dependencies, and no update without a release date, see below |
| `undated.json5` | No automerge for images from registries without release dates (GHCR, Quay…), see [Undated images](#undated-images) |
| `kubernetes.json5` | File patterns for the `kubernetes`, `argocd` and `flux` managers |
| `annotated.json5` | Updates any version that has a `# renovate:` comment above it, see below |
| `.renovaterc.json5` | Renovate config for this repo itself (not a preset) |

## Automerge

Minor, patch and digest updates are automerged once they are 7 days old
([Release age](#release-age-7-days)). Majors, 0.x minors, most database minors
([Databases](#databases)), date versions ([CalVer](#date-versions-calver)) and
[undated images](#undated-images) get a PR that is merged by hand.

Automerged updates go through a PR that GitHub merges once the required checks pass
(`:automergePr`). Renovate can also push straight to the base branch without a PR
(`:automergeBranch`), but in practice it opened a PR every time anyway: every Renovate update
in `docker_containers` reached the base branch through a PR, most likely because the branch
requires PRs and refuses the direct push. With a PR, your CI runs before the merge, as long as
the workflow runs on `pull_request`.

GitHub's auto-merge only waits for **required** checks. To make a lint job block a bad
update, mark it as required in the branch protection rules (only for a workflow that runs on
every PR, not one limited to some `paths`).

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
    "ignorePresets": ["local>ansasi/renovate-config//calver.json5"]
}
```

Use the same prefix as in your `extends` (`local>` or `github>`), followed by `//` and the file
name. Renovate turns the relative paths in `default.json` into that form before it checks
`ignorePresets`, so `./calver.json5` or the old `:calver.json5` form won't match.

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

## Release age: 7 days

Every update waits until the new version is 7 days old: every datasource, majors and digests
included (`minimumReleaseAge`). Renovate proposes the newest version that is old enough and
doesn't create a branch for anything newer (`internalChecksFilter: strict`); the held updates
are listed under "Pending Status Checks" in the Dependency Dashboard.

### Why

A hijacked maintainer account or a compromised build pipeline can publish a poisoned release.
Most are spotted and pulled within days, and Renovate is exactly what would pull such a release
in quickly. A week covers most of them; databases already waited 7 days for another reason
([Databases](#databases)).

### Only with a release date

Renovate can only hold a release when the registry publishes its date:

| Source | Release date |
|---|---|
| Docker Hub | yes, but only for images with up to about 1,000 tags without a Docker Hub login (see below) |
| GHCR, Quay, `lscr.io`, ECR and other container registries | no: see [Undated images](#undated-images) |
| Helm chart repositories (`index.yaml`) | yes |
| Helm charts stored as OCI images (on GHCR, Quay…) | no: they are container images, see [Undated images](#undated-images) |
| GitHub releases and tags (GitHub Actions too) | yes |
| npm, PyPI, crates.io, Go modules | yes |

**Docker Hub with many tags.** Without a login, Docker Hub only lists the first 10 pages (about
1,000 tags) and then answers "pagination offset too large for anonymous requests". Renovate
then reads the tags from the registry instead, which has no dates. Images with years of tags
are affected, for example `linuxserver/*`, `traefik`, `portainer/portainer-ce`, `postgres`,
`redis`, `grafana/grafana` and `searxng/searxng`. A login doesn't help: Docker Hub's API ignores
it and only accepts a bearer token that expires after 10 minutes, shorter than a Renovate run.
So a Docker Hub image without a date is **held** (`minimumReleaseAgeBehaviour:
timestamp-required`, in `undated.json5`): its updates are listed under "Pending Status Checks" in
the Dependency Dashboard and get a PR only when ticked, merged by hand. Without that rule they
would be raised without any wait and automerged.

### Urgent security fixes

The wait only limits what Renovate raises and automerges by itself; you can merge any update
by hand at any time. For a fix you need now:

- **Held by the wait:** tick it under "Pending Status Checks" in the Dependency Dashboard to
  get the PR right away, review it, and merge it.
- **Not raised at all** (a container image whose fix Renovate doesn't know about): change the
  version in a small PR yourself.
- **Vulnerability alerts:** for the ecosystems GitHub or OSV track (npm, PyPI, Go, crates,
  GitHub Actions…), Renovate raises a fix PR without the wait, labelled `security`. It needs
  the repo's *Dependency graph* and *Dependabot alerts* turned on, and read access to
  Dependabot alerts for the Renovate app. Container images and Helm charts are not covered.

## Package registries

npm, PyPI, crates.io and Go modules wait 7 days like everything else, with two extras.

### Why

These registries have shipped malware many times, usually when an attacker takes over a
maintainer's account and publishes a poisoned version of a popular package (`event-stream`,
`ua-parser-js`, `ultralytics`, the 2025 `chalk`/`debug` and "Shai-Hulud" npm worms...). Most are
spotted and pulled within hours or a few days. Renovate is exactly what would pull such a
version in quickly. Even a branch that never gets merged is a risk, because CI installs the
package and npm/PyPI install scripts run there, with the CI secrets.

### How

- npm, PyPI and crates.io use Renovate's own `security:minimumReleaseAge*` presets. Go has no
  built-in preset, so it has its own rule.
- For npm, Renovate also passes `--before=<the wait>` to npm when it updates the lock file, so
  new versions of *indirect* dependencies are held too.
- A version with no release date is held, not raised (`timestamp-required`), the opposite of
  the Docker setting in [Other settings](#other-settings). All four registries publish release
  dates, so this only matters for a private mirror that doesn't.
- Renovate's presets set 3 days; `supplyChain.json5` keeps them at 7 like the rest.

### Not covered

- **Pin updates** (`:pinDevDependencies` turns `^1.2.0` into `1.2.3`) and lock file maintenance
  skip the wait. Renovate adds a warning to those PRs.
- **Indirect dependencies outside npm.** Only npm (and Poetry) are told about the wait. Other
  lock files can still resolve a brand-new indirect dependency. If your package manager has its
  own setting (pnpm `minimumReleaseAge`, Yarn `npmMinimalAgeGate`, uv `exclude-newer`), set it
  too.
- **Installs outside Renovate:** an `npm install` you run by hand or in CI without a lock file.

## Undated images

Images from a registry other than Docker Hub (`ghcr.io/…`, `quay.io/…`, `lscr.io/…`,
including Helm charts stored as OCI images) have no release date, so the 7-day wait can't hold
them. They are raised as soon as they appear, with the `undated` label, and never automerged:
merge them by hand once you are happy with the release.

Many projects publish the same image on Docker Hub too. Compare the digests of a tag on both
registries: when they are equal, switching the image to Docker Hub brings back the wait and the
automerge. The `undated` rule matches any image whose name starts with a registry host other
than Docker Hub's (`docker.io`, `index.docker.io`, `registry-1.docker.io`).

A repo rule that sets `automerge: true` for such an image overrides this, because repo rules run
after the presets. Only do that for an image you'd accept without any wait.

## 0.x versions

Under SemVer, anything below 1.0 can break at any time, so `0.4 → 0.5` is really a major
update. 0.x minor updates are not automerged and get the `type/major` label. 0.x patches
(`0.4.1 → 0.4.2`) are still automerged.

## Annotated versions

Some versions live where no Renovate manager looks: a Dockerfile `ARG`, a `.env` file, a shell
script, a Helm value. Put a comment on the line above to tell Renovate where to find updates:

```dockerfile
# renovate: datasource=github-releases depName=kubernetes-sigs/kustomize
ARG KUSTOMIZE_VERSION=v5.4.3
```

```yaml
app:
  # renovate: datasource=docker depName=ghcr.io/home-operations/radarr
  tag: 5.8.3.8933
```

- `datasource` and `depName` are required, then optionally `packageName=` (when the name to
  look up differs from `depName`) and `versioning=`, in that order.
- The comment must start its own line (indenting is fine). Examples inside other comments, like
  `//   # renovate: ...` in a JSON5 file, are ignored.
- The version is the value after the first `:` or `=` on the next line. Quotes and YAML
  anchors (`version: &v "1.2.3"`) are fine.
- If the GitHub tags are `v1.2.3` and your value is `1.2.3`, Renovate keeps it without the `v`
  (with the default versioning).
- These updates follow the same rules as the rest: automerge, labels, databases, CalVer.

## Other settings

Set in `default.json`:

- **`minimumReleaseAgeBehaviour: timestamp-optional`.** Since Renovate 42, an update without a
  release date never passes `minimumReleaseAge` and is held forever. GHCR, Quay and ECR don't
  publish release dates, so without this setting their images would never update. The
  downside: those images skip the 7-day wait, so they are never automerged
  ([Undated images](#undated-images)). Package registries are the exception, see
  [above](#package-registries).
- **`rebaseWhen: conflicted`.** A PR is only rebased when it conflicts with the base branch,
  not every time the base branch moves: fewer CI runs. The repos' base branches don't require
  PRs to be up to date before merging; with that rule on, automerge would wait for a rebase.
- **`vulnerabilityAlerts`** add the `security` label; Renovate's defaults already skip the
  wait and the schedule for them ([Urgent security fixes](#urgent-security-fixes)).
- **`abandonments:recommended`.** Packages with no release for a year are flagged as abandoned
  in the Dependency Dashboard.
- **`:configMigration`.** When a repo's own Renovate config uses outdated options, Renovate
  opens a PR to update it.
- **`helpers:pinGitHubActionDigestsToSemver`.** GitHub Actions are pinned to a digest, and the
  comment shows the full version (`# v4.2.1`) instead of the short tag (`# v4`).
- `:pinDevDependencies`, `:timezone(Europe/Madrid)`, no Renovate comments when a PR is edited or
  closed (`suppressNotifications`).

## Credits

This config started as a fork of
[home-operations/renovate-config](https://github.com/home-operations/renovate-config).
Thanks to the Home Operations folks for the original presets. The `LICENSE` keeps their
MIT copyright notice.
