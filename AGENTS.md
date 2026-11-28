# Coding agents

## What this repo is

This repo (`firefliesai/mcp-registry`) is a public fork of `docker/mcp-registry`.
It holds the curated Docker MCP server catalog under `servers/` and Go CLI tooling under `cmd/`, `pkg/`, and `internal/`.
Fireflies' catalog entry lives at `servers/firefly/` (`server.yaml`, `readme.md`, `tools.json`).

There are two Go modules in this repo:

- Root module (`go.mod`, `go 1.24.0`, `module github.com/docker/mcp-registry`) for the registry CLI tools.
- Security reviewer module (`agents/security-reviewer/go.mod`, `go 1.25.3`) for the containerized reviewer and proxy.

## Prerequisites and setup

- Go 1.24+ for the root module. Go 1.25.3+ for `agents/security-reviewer` (Go fetches it automatically when `GOTOOLCHAIN=auto`). CI runs Go `1.26.10`.
- [Task](https://taskfile.dev/) v3 for `Taskfile.yml` wrappers. Every Go task can also run directly with `go run ./cmd/<name>` or `go test ./...`.
- Docker Engine, Docker Buildx, and Docker Compose for building local server images (`task build`, `task create`, `task clean`) and running `agents/security-reviewer`.
- The `docker mcp` CLI plugin is only needed for `task import` and `task reset`.
- Node.js with npm (`npx`) for `task validate`, which runs `npx --yes prettier --check` on `server.yaml` (downloading Prettier on first use).
- No `load-env`, root package install step, or pre-commit hooks are configured in this repo.

## Build, vet, test, and validate

Run commands from the repo root unless a command changes directory.
The commands in this section, except `task clean`, were verified locally without credentials.

Root module build, vet, and unit tests (only `cmd/validate/main_test.go` currently has tests):

```sh
go build ./...
go vet ./...
go test ./...
```

The same unit tests via Task:

```sh
task unittest
```

Security reviewer Go module build, vet, and tests:

```sh
cd agents/security-reviewer && go build ./... && go vet ./... && go test ./...
```

Validate a single server definition under `servers/<name>/server.yaml` (fetches icon URLs over the network, and queries the GitHub API for local server license checks):

```sh
task validate -- --name firefly
```

Build a server image and list tools (skipped automatically for `remote` and `poci` servers; local servers require Docker, and `--pull-community` pulls images outside `mcp/`):

```sh
task build -- --tools firefly
```

Generate a local test catalog at `catalogs/<name>/catalog.yaml` (`catalogs/` is gitignored):

```sh
task catalog -- firefly
```

To clean generated catalog and Docker artifacts for a server, run `task clean -- firefly` (replacing `firefly` with the server name). Note that `cmd/clean` also runs `docker builder prune --force` and `docker image prune --force` globally on the host, so this command was not run locally.

There is no separate Go linter config (such as `.golangci.yml`) in this repo. YAML formatting is checked via Prettier inside `task validate`.

## Commands and workflows that need credentials

- `GITHUB_TOKEN` (optional in code, needed to avoid GitHub API rate limits or access private repos) is read by `pkg/github/github.go`, `cmd/create/main.go`, and `cmd/build/main.go`. It applies to `task validate` on local servers, `task create`, `task build`, `task wizard`, `task catalog` on local servers, and `task ci -- update-pins`. When unset, the client falls back to unauthenticated requests.
- `ANTHROPIC_API_KEY` (for `--agent claude`, the default) or `OPENAI_API_KEY` (for `--agent codex`) is required by `task security-reviewer` (`cmd/security-reviewer/main.go` and `agents/security-reviewer/compose.yml`).
- Upstream GitHub Actions secrets and variables (`REVIEWER_REPOSITORY`, `MCP_REGISTRY_BOT_APP_ID`, `MCP_REGISTRY_BOT_PRIVATE_KEY`) are used by `.github/workflows/security-review-trigger.yaml`, `update-pins.yaml`, and `auto-merge-pins.yaml`. Because `REVIEWER_REPOSITORY` is not configured in `firefliesai/mcp-registry`, the `Trigger Security Review` workflow fails on same-repo PRs targeting `main` in this fork (external-fork PRs and PRs targeting other branches skip it). It is not a required check.

## Repo rules and CI

- The `firefliesai` org ruleset on `main` blocks branch deletion (`deletion`) and force-push (`non_fast_forward`), allows only merge commits (`allowed_merge_methods: ["merge"]`), requires 1 approving review (`required_approving_review_count: 1`, with `require_extra_approval_for_unattributed_changes: true`), and enforces org `vuln-scan.yaml` and `CodeQL` checks.
- `.github/workflows/ci.yaml` (`Code and Server Validation`) runs on every `pull_request`. It builds validation binaries from `main`, runs `go test ./...` on the PR branch, and runs `scripts/ci-validation.sh` on modified `servers/<name>/` directories.
- Keep `servers/firefly/` directory names, OAuth provider names, and secret keys stable when updating display metadata so existing users keep their enabled server and saved credentials.
- `.github/PULL_REQUEST_TEMPLATE.md` is for new MCP server submissions to upstream `docker/mcp-registry`. Use a concise summary and testing section for docs or tooling PRs.
- Do not commit `catalogs/` (already in `.gitignore`).

## Starting points

- `Taskfile.yml` defines the CLI wrappers (`create`, `build`, `catalog`, `wizard`, `remote-wizard`, `validate`, `clean`, `ci`, `import`, `reset`, `unittest`, `security-reviewer`).
- `servers/<name>/server.yaml` defines each MCP server entry (`servers/firefly/` is Fireflies' entry).
- `cmd/` contains the Go CLI entrypoints (`validate`, `build`, `catalog`, `clean`, `create`, `wizard`, `remote-wizard`, `ci`, `security-reviewer`).
- `pkg/` contains shared registry packages (`servers`, `catalog`, `github`, `hub`), and `internal/` contains `licenses` and `mcp` helpers.
- `scripts/ci-validation.sh` runs the validate, build, catalog, and clean pipeline for changed servers in CI.
- `agents/security-reviewer/` holds the separate Go module, Dockerfile, and Compose manifest for automated security reviews.
- `CONTRIBUTING.md`, `add_mcp_server.md`, and `docs/configuration.md` document the `server.yaml` schema and server submission flow.
