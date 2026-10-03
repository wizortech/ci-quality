# CI/Quality Workflows

Reusable GitHub Actions workflows for code quality, security scanning, and build/test automation across Node, Java, and general security tooling.

## Workflows

### security.yml

Performs security scanning with gitleaks, Semgrep, and Trivy.

**Inputs:**
- `working-directory` (string, default: `.`) — Directory to scan relative to repository root
- `enforce` (boolean, default: `true`) — If false, Semgrep and Trivy only report; gitleaks always blocks

**Scanners:**
- **gitleaks** — Detects secrets in committed code (always blocks on findings)
- **Semgrep CE** — Static analysis for code quality and security patterns
- **Trivy** — Scans for vulnerabilities, misconfigurations, and secrets in dependencies and configs

### node.yml

Installs dependencies and runs scripts for Node.js projects (npm or bun).

**Inputs:**
- `working-directory` (string, default: `.`) — Directory containing package.json
- `package-manager` (string, default: `npm`) — Package manager to use: `npm` or `bun`
- `node-version` (string, default: `""`) — Node.js version to install (if using npm)
- `node-version-file` (string, default: `""`) — File containing Node.js version (if using npm)
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

## Pinned Action and Image Versions

All actions and container images are pinned for reproducibility and security:

- **GitHub Actions** are referenced by exact commit SHA with version tags in comments (e.g., `actions/checkout@df4cb1c069e1874edd31b4311f1884172cec0e10 # v6.0.3`)
- **Container images** are referenced by image tag and full SHA256 digest (e.g., `image:tag@sha256:...`)

This ensures that builds are deterministic and prevents supply-chain compromises from unexpected action/image updates.

## License

MIT — Copyright Wizortech
