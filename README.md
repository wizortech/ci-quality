# CI/Quality Workflows

Reusable GitHub Actions workflows for code quality, security scanning, and build/test automation across Node, Java, and general security tooling.

## Workflows

All workflows run by default on `self-hosted`, i.e. any of the organization's 5 ARM64 runners
(`arm64-sp` and `arm64-sp-sem-kubeconfig`). None of them needs kubeconfig or registry credentials, and
pinning them to the 2 runners without kubeconfig made those the bottleneck while the 3 `arm64-sp` sat
idle (measured in `wizortech/deploy-workflows`, ADR 0006). Override with the `runner` input.

### security.yml

Performs security scanning with gitleaks, Semgrep, and Trivy.

**Inputs:**
- `working-directory` (string, default: `.`) — Directory to scan relative to repository root
- `enforce` (boolean, default: `true`) — If false, Semgrep and Trivy only report; gitleaks always blocks
- `block-critical` (boolean, default: `false`) — Blocks on any Trivy CRITICAL finding (vuln, misconfig, secret) even with `enforce: false`, and fails if Trivy cannot run (e.g. Maven 429). Meant for repos whose CRITICAL baseline is zero while HIGH/Semgrep are still being cleaned up

**Scanners:**
- **gitleaks** — Detects secrets in committed code (always blocks on findings)
- **Semgrep CE** — Static analysis for code quality and security patterns
- **Trivy** — Scans for vulnerabilities, misconfigurations, and secrets in dependencies and configs

### gitleaks.yml

Only the gitleaks job of `security.yml`, for deploy pipelines (Semgrep and Trivy run on PRs and nightly).

**Inputs:** `working-directory` (default `.`), `runner`.

### alerta-telegram.yml

Sends a failure message (repository, workflow and run link) to the platform Telegram bot.
Meant for nightly workflows, in a job with `needs: [...]` and `if: failure()`.

**Secrets:** none. It reads `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID` from Doppler `platform/ci`
with the read-only service token in the self-hosted runners' environment (`DOPPLER_TOKEN` in each
runner's `.env`). Organization secrets do not reach private repositories on GitHub Free, so the
token lives on the runner instead of in 11 repositories. `DOPPLER_TOKEN` is still accepted as an
optional secret during the transition.

## Actions

### doppler-ci

`uses: wizortech/ci-quality/.github/actions/doppler-ci@v1.2.0` (with `id: doppler`) reads Doppler
`platform/ci` with the runner's `DOPPLER_TOKEN`, masks every value and exposes each key as an
output (`steps.doppler.outputs.CLOUDFLARE_API_TOKEN`, `HOSTINGER_*`, `GOOGLE_MAPS_API_KEY`,
`TELEGRAM_*`). Only works on the org's self-hosted runners. Optional input `token` overrides the
runner token.

### node.yml

Installs dependencies and runs scripts for Node.js projects (npm or bun).

**Inputs:**
- `working-directory` (string, default: `.`) — Directory containing package.json
- `package-manager` (string, default: `npm`) — Package manager to use: `npm` or `bun`
- `node-version` (string, default: `""`) — Node.js version to install (required with npm; optional with bun, where it installs Node for tools like Vitest and Prisma)
- `node-version-file` (string, default: `""`) — File containing Node.js version (required with npm unless node-version is set; optional with bun)
- `bun-version` (string, default: `""`) — Bun version to install (if using bun)
- `scripts` (string, **required**) — Space-separated list of npm/bun scripts to run (e.g., `lint typecheck test`)

### java.yml

Sets up Java environment and runs Maven build/test commands.

**Inputs:**
- `working-directory` (string, default: `.`) — Directory containing pom.xml
- `java-version` (string, default: `"25"`) — Java version to use (Temurin distribution)
- `command` (string, default: `./mvnw -B verify`) — Maven command to execute

## Usage Example

```yaml
name: CI
on: [push, pull_request]

jobs:
  security:
    uses: wizortech/ci-quality/.github/workflows/security.yml@v1
    with:
      working-directory: .
      enforce: true

  node-quality:
    uses: wizortech/ci-quality/.github/workflows/node.yml@v1
    with:
      working-directory: ./apps/web
      package-manager: npm
      scripts: lint typecheck test

  java-build:
    uses: wizortech/ci-quality/.github/workflows/java.yml@v1
    with:
      working-directory: ./backend
      java-version: "21"
      command: ./mvnw -B clean verify
```

### Nightly example

```yaml
name: Nightly
on:
  schedule:
    - cron: "0 6 * * *" # 03:00 BRT
  workflow_dispatch:

jobs:
  security:
    uses: wizortech/ci-quality/.github/workflows/security.yml@v1
  alerta:
    needs: [security]
    if: failure()
    uses: wizortech/ci-quality/.github/workflows/alerta-telegram.yml@v1
    secrets:
      DOPPLER_TOKEN: ${{ secrets.DOPPLER_TOKEN }}
```

## Self-test

`selftest.yml` runs `gitleaks.yml` and `security.yml` against this repository on every PR, on a
GitHub-hosted runner: this repository is public and the organization's self-hosted runners do not
serve public repositories (and must not: fork PRs would run on machines with sudo). Keep it green
before moving the `v1` tag; the self-hosted runners are proven by the first private app PR that
calls the new version.

## Pinned Action and Image Versions

All actions and container images are pinned for reproducibility and security:

- **GitHub Actions** are referenced by exact commit SHA with version tags in comments (e.g., `actions/checkout@df4cb1c069e1874edd31b4311f1884172cec0e10 # v6.0.3`)
- **Container images** are referenced by image tag and full SHA256 digest (e.g., `image:tag@sha256:...`)

This ensures that builds are deterministic and prevents supply-chain compromises from unexpected action/image updates.

## License

MIT — Copyright Wizortech
