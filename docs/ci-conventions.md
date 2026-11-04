# CI conventions

The rules a repository follows when it calls the workflows of `ESchuderer/reusable-workflows`, for the owner and for agents that set up or change a repository.

## Required check

- Every repository has exactly one required check, the job `check` in `ci.yml`, built with `re-actors/alls-green`. The ruleset requires it with the source GitHub Actions (integration id `15368`).
- `check` runs with `if: always()` and lists every gate and product job in `needs`. A skipped job fails `check` unless it is named in `allowed-skips`.
- Affected-only builds: a monorepo computes the changed areas in an early job (`dorny/paths-filter` or `pnpm --filter "...[origin/main]"`), runs area jobs by `if:` and names them in `allowed-skips`:

  ```yaml
  check:
    if: always()
    needs: [gates, changes, app, firmware]
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: re-actors/alls-green@<sha> # v1.3.0
        with:
          jobs: ${{ toJSON(needs) }}
          allowed-skips: app, firmware
  ```
- No workflow-level `paths:` filter on a workflow that reports a required check: a skipped workflow leaves the check pending.
- A required check that never reported blocks every merge, so the ruleset with `check` is applied after `ci.yml` ran once.
- CodeQL stays out of `needs`: the ruleset's code scanning rule enforces its result.

## Events, permissions, timeouts, concurrency

- `ci.yml` triggers on `push` to `main`, `pull_request` with types `opened, edited, reopened, synchronize`, and `merge_group`. `edited` reruns the title check after a title change. Steps that need a pull request skip on the other events and their job still reports.
- Every workflow starts with `permissions: {}` and grants per job what that job needs. A called workflow can only lower the permissions its caller grants, so the caller grants the union the called jobs need (the table in the README).
- Every job sets `timeout-minutes` (gates 5, CodeQL 30, weekly release 120).
- Pull request runs cancel superseded runs; runs on `main` never cancel: `cancel-in-progress: ${{ github.event_name == 'pull_request' }}`.
- `actions/checkout` runs with `persist-credentials: false`.
- Secrets are passed by name (`secrets: app-private-key: ...`); `secrets: inherit` does not cross owners.
- The weekly release runs `cron: '0 14 * * 1'` with `timezone: Europe/Berlin`. A repository may drop the schedule and run it by hand.

## Pins and versions

- Every `uses:` names a full 40-character commit SHA with the version as a comment: `uses: ESchuderer/reusable-workflows/.github/workflows/gates.yml@<sha> # v0.1.0`. No tag, no branch.
- The SHA of a release: `gh api repos/ESchuderer/reusable-workflows/commits/v0.1.0 --jq .sha`.
- Releases of this repository are immutable; a published tag cannot move.
- Dependabot keeps pins current with a cooldown of 3 days in every ecosystem (`templates/dependabot.yml`). The pins of `ESchuderer/reusable-workflows` have their own group, so they arrive as their own pull request.
- `pin-auto-merge.yml` turns on auto-merge for that pull request when it changes nothing but those pins; the ruleset merges it once `check` is green. A pull request that also changes any other dependency waits for the owner.
- Accepted risk: a compromised release of `ESchuderer/reusable-workflows` reaches every caller without a review, after the 3-day cooldown. Immutable releases, SHA pins, zizmor and the rule that only the owner changes workflow files limit it.
- zizmor's `dependabot-cooldown` audit asks for 7 days. `gates.yml` writes a zizmor config with 3 days when the repository has none; a repository with its own `zizmor.yml` sets `rules.dependabot-cooldown.config.days: 3` there.
- Auto-merge turned on with `GITHUB_TOKEN` merges as `github-actions[bot]`, and GitHub starts no workflow for events caused by that token, so the pin merge itself runs no push workflow on `main`; the next push runs CI and release-please over it (expected from GitHub's documentation, not yet observed).
- Manual bumps, which Dependabot does not see: the actionlint image digest (in `gates.yml` and in the `templates` job of `ci.yml`) and the zizmor version (in `gates.yml`).

## Inputs

`gates.yml`:

| Input | Default | Use |
|---|---|---|
| `license-allow-list` | `MIT, Apache-2.0, BSD-2-Clause, BSD-3-Clause, ISC, 0BSD, CC0-1.0, Unlicense, MPL-2.0, LGPL-2.1-or-later, LGPL-3.0-or-later` | SPDX ids a new dependency may carry. A repository adds GPL or AGPL only on purpose, by passing the full list with them added. |
| `fail-on-severity` | `high` | lowest vulnerability severity that fails dependency review |
| `pr-title-types` | `feat fix perf refactor docs test build ci chore revert style`, one per line | Conventional Commits types a title may use |
| `typos-config` | empty | path to a typos config; empty lets typos find `_typos.toml` |
| `check-links` | `true` | offline check of links between local Markdown files |

Dependency review and GitHub Actions: GitHub Actions dependencies carry no license data that dependency review reads, so an action bump passes the allow-list without an exception (see the evidence in the pull request that introduced `gates.yml`).

`codeql.yml`:

| Input | Default | Use |
|---|---|---|
| `languages` | `'["actions"]'` | JSON list of CodeQL languages, for example `'["actions", "javascript-typescript"]'` or `'["python"]'` |
| `build-mode` | `none` | `none`, `autobuild` or `manual` for every language of the call |
| `build-command` | empty | shell command that builds the code when `build-mode` is `manual` |

A repository with compiled code calls `codeql.yml` twice: once with `build-mode: none` for `actions` and the interpreted languages, once with `languages: '["c-cpp"]'`, `build-mode: manual` and its build command. The C and C++ build with PlatformIO (`build-command: pio run`) is unverified: CodeQL's tracing of the PlatformIO cross-compiler has not been tested.

`release-weekly.yml`: input `app-client-id` (empty fails the job), input `release-workflow` (default `release.yml`), secret `app-private-key`.

`pin-auto-merge.yml`: no inputs.

## Allowlist

An owner or repository with the Actions policy `selected` and SHA pinning required allows GitHub-owned actions and names:

- `ESchuderer/reusable-workflows/.github/workflows/*@*` (one entry for every shared workflow; whether the wildcard matches a workflow path is not yet tested)
- the third-party actions used inside the shared workflows: `zizmorcore/zizmor-action@*`, `crate-ci/typos@*`, `lycheeverse/lychee-action@*`, `amannn/action-semantic-pull-request@*`, `dependabot/fetch-metadata@*`
- the third-party actions of the templates the repository uses: `re-actors/alls-green@*`, `googleapis/release-please-action@*`, `ossf/scorecard-action@*`, and `dorny/paths-filter@*` where an area filter runs

## Checklist for a new public repository

1. Add it to the repository list of its owner in `ESchuderer/iac`, run `tofu plan`, read it, apply. This sets the merge settings, the Actions policy and allowlist, the workflow token defaults and the ruleset.
2. Set the three fields the provider cannot set, and read each back:
   - private vulnerability reporting on: `gh api -X PUT repos/<owner>/<repo>/private-vulnerability-reporting`, check with `gh api repos/<owner>/<repo>/private-vulnerability-reporting`
   - immutable releases on: `gh api -X PUT repos/<owner>/<repo>/immutable-releases`, check with `gh api repos/<owner>/<repo>/immutable-releases`
   - CodeQL default setup off: `gh api -X PATCH repos/<owner>/<repo>/code-scanning/default-setup -f state=not-configured`, check with `gh api repos/<owner>/<repo>/code-scanning/default-setup --jq .state`
3. Copy the templates and set the release SHA (section Pins and versions). Add `_typos.toml` with `[default] locale = "en-us"` for US spelling.
4. A repository that releases: set the variable `RELEASE_APP_CLIENT_ID` and the secret `RELEASE_APP_PRIVATE_KEY` of its owner's release GitHub App, and add `release-please-config.json` and `.release-please-manifest.json`.
5. Open the first pull request. Once `check` has reported, the ruleset that requires it is applied (step 1 again if it was held back).

## Private repositories

Private repositories are set by hand from the same checklist and are never named in `ESchuderer/iac`. On GitHub Free a private repository has no rulesets, so CI reports and cannot block a merge, and CodeQL, dependency review and secret scanning do not run. Each private repository writes this deviation down with its substitute gates:

| Gate | Substitute |
|---|---|
| CodeQL for workflows | actionlint and zizmor |
| CodeQL for Python | ruff with `select = ["ALL"]` and basedpyright `recommended` |
| CodeQL for TypeScript | Oxlint with typed rules as errors and `tsc` strict |
| CodeQL for PHP (not analyzed by CodeQL) | `php -l` and PHPStan |
| Dependency review | Dependabot version updates and alerts, hash-pinned lockfiles |
| Secret scanning | gitleaks over the full history |
| Text | typos and lychee offline |

## What agents must not change

Agent tokens carry no Workflows permission: an agent cannot push a change under `.github/workflows/`, and only the owner changes workflow files and the files they run. Agents do not change rulesets, the Actions policy, the allowlist or repository settings; those change through `ESchuderer/iac`, applied by the owner.

## Product-side tools the gates assume

The gates cover workflows, text, dependencies and titles. Each product repository runs its own build, test, lint and format jobs and lists them in `needs` of `check`:

- pnpm workspaces for JavaScript and TypeScript, without Turborepo; affected-only runs through pnpm's filter or `dorny/paths-filter`
- Oxlint as the linter, with import rules for boundaries and cycles, plus knip
- Biome as the formatter
- Stryker mutation tests weekly in the product repository, never as a pull-request gate
- no hosted code-quality scan as a gate
