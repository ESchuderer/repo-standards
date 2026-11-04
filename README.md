# reusable-workflows

Shared GitHub Actions workflows that give a repository its pull-request gates, CodeQL scanning, a weekly release merge and automatic merging of its own pin updates. A repository calls them by full commit SHA from an immutable release, so a CI change is made once here and reaches every caller as a Dependabot pull request.

| Workflow | What it does | Caller grants |
|---|---|---|
| `gates.yml` | actionlint and zizmor on the workflows, typos and an offline link check on the text, dependency review with a license allow-list, Conventional Commits pull request titles | `contents: read`, `pull-requests: read` |
| `codeql.yml` | CodeQL advanced setup for a JSON list of languages | `security-events: write`, `contents: read`, `actions: read` |
| `release-weekly.yml` | merges the open release-please pull request with the release GitHub App | `contents: read`, `pull-requests: write`, `actions: write` |
| `pin-auto-merge.yml` | turns on auto-merge for Dependabot pull requests that change only the pins of this repository | `contents: write`, `pull-requests: write` |

## Use

1. Copy the files you need from [`templates/`](templates/) into the repository (`dependabot.yml` into `.github/`, the rest into `.github/workflows/`).
2. Replace `<release-sha> # <version>` with the commit of the latest release: `gh api repos/ESchuderer/reusable-workflows/commits/vX.Y.Z --jq .sha`.
3. Follow the checklist in [docs/ci-conventions.md](docs/ci-conventions.md): required check `check`, Actions allowlist, the three settings the API tooling cannot set, the release App.

Inputs, defaults and the rules a caller follows are in [docs/ci-conventions.md](docs/ci-conventions.md).

## Versions

Releases follow semantic versioning and are immutable. A removed or renamed input, a changed default or a changed permission need is a breaking change. Callers pin by full commit SHA with the version as a comment; there is no floating major tag.

## License

See [LICENSE](LICENSE).
