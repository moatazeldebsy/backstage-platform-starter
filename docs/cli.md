# idp CLI

The `idp` CLI is the golden-path command-line tool for scaffolding services and QA test suites. It uses the Backstage Scaffolder API when the portal is reachable, and falls back to local file generation when offline.

## Installation

```bash
# Build and install to $(go env GOPATH)/bin/idp
make cli-build    # builds to ./bin/idp
make cli-install  # installs to $(go env GOPATH)/bin/idp
```

Confirm the install:

```bash
idp --help
```

---

## `idp scaffold service`

Scaffold a new microservice. When Backstage is reachable the full golden path runs (GitHub repo created, service registered in catalog, GitOps PR opened, TechDocs generated). Otherwise files are generated locally under `services/<name>/`.

### Usage

```
idp scaffold service --name <name> --type <type> [flags]
```

### Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--name` | *(required)* | Service name — lowercase alphanumeric + hyphens |
| `--type` | `nodejs` | Runtime: `nodejs` \| `python` \| `go` |
| `--namespace` | `services` | Kubernetes namespace |
| `--owner` | `group:default/platform-team` | Backstage catalog owner ref |
| `--description` | | Short description used by Backstage template |
| `--local` | `false` | Skip Backstage API, generate files locally |
| `--backstage-url` | `http://backstage.idp.local` | Backstage base URL |

### Token resolution (Backstage API mode)

The CLI resolves the auth token in this priority order:

1. `--token` flag
2. `BACKSTAGE_TOKEN` environment variable
3. `BACKSTAGE_AUTH_SECRET` in `local/backstage/.env`
4. First static `externalAccess` token in `backstage/app-config.local.yaml`

### GitHub org resolution

Set `GITHUB_ORG` (or `GH_ORG`) in your shell or in `local/.env`. If unset, the CLI warns and uses the placeholder `YOUR_GITHUB_ORG`.

### Examples

```bash
# Node.js service (auto-detects Backstage)
idp scaffold service --name order-svc --type nodejs

# Python FastAPI service, local generation (offline / pre-Backstage)
idp scaffold service --name data-pipeline --type python --local

# Go service in a custom namespace
idp scaffold service --name inventory-svc --type go --namespace backend

# Explicit description and owner
idp scaffold service --name billing-svc --type nodejs \
  --description "Billing microservice" \
  --owner group:default/payments-team
```

### Local output (`--local`)

```
services/<name>/
├── Dockerfile
├── catalog-info.yaml
├── helm-values.yaml
├── helm-values-local.yaml
├── helm-values-dev.yaml
├── helm-values-staging.yaml
├── README.md
└── src/            # language-specific source files
```

---

## `idp scaffold test-suite`

Scaffold a QA / testing suite for an existing service. Supported types cover the full testing pyramid from unit contracts to chaos engineering.

### Usage

```
idp scaffold test-suite --name <name> --type <type> --service <service> [flags]
```

### Common flags

| Flag | Default | Description |
|------|---------|-------------|
| `--name` | *(required)* | Suite name — lowercase alphanumeric + hyphens |
| `--type` | *(required)* | Suite type (see table below) |
| `--service` | *(required)* | Target service name |
| `--namespace` | `services` | Kubernetes namespace of the target service |
| `--owner` | `group:default/platform-team` | Backstage catalog owner ref |
| `--description` | | Short description |
| `--local` | `false` | Skip Backstage API, generate files locally |
| `--backstage-url` | `http://backstage.idp.local` | Backstage base URL |

### Suite types and type-specific flags

#### `playwright` — E2E browser tests

```bash
idp scaffold test-suite --name my-e2e --type playwright --service my-svc
```

No extra flags. Generates a Playwright TypeScript suite with GitHub Actions workflow.

---

#### `k6` — Performance / load tests

```bash
idp scaffold test-suite --name my-load --type k6 --service my-svc \
  --vus 50 --duration 5m --p95 300
```

| Flag | Default | Description |
|------|---------|-------------|
| `--vus` | `10` | Number of virtual users |
| `--duration` | `30s` | Load test duration (e.g. `1m`, `5m`) |
| `--p95` | `500` | p95 latency threshold in ms |

---

#### `pact` — Consumer-driven contract tests

```bash
idp scaffold test-suite --name my-contracts --type pact --service my-svc \
  --consumer frontend --provider my-svc --broker-url https://YOUR_ORG.pactflow.io
```

| Flag | Default | Description |
|------|---------|-------------|
| `--consumer` | `<name>-consumer` | Consumer name |
| `--provider` | `<service>` | Provider name |
| `--broker-url` | `https://YOUR_ORG.pactflow.io` | Pact Broker URL |

---

#### `newman` — API (Postman) collection tests

```bash
idp scaffold test-suite --name my-api --type newman --service my-svc
```

No extra flags. Generates a Newman runner with a Postman collection template.

---

#### `zap` — DAST security scan (OWASP ZAP)

```bash
idp scaffold test-suite --name my-sec --type zap --service my-svc \
  --scan-type baseline --fail-risk High
```

| Flag | Default | Description |
|------|---------|-------------|
| `--scan-type` | `baseline` | Scan type: `baseline` \| `full` \| `api` \| `graphql` |
| `--openapi-url` | `http://localhost:8080/openapi.json` | OpenAPI spec URL |
| `--fail-risk` | `High` | Minimum risk level to fail: `Low` \| `Medium` \| `High` |

---

#### `datadog` — Synthetic monitoring

```bash
idp scaffold test-suite --name my-synthetics --type datadog --service my-svc \
  --dd-site datadoghq.eu
```

| Flag | Default | Description |
|------|---------|-------------|
| `--dd-site` | `datadoghq.eu` | Datadog site (`datadoghq.com`, `datadoghq.eu`, etc.) |

---

#### `visual` — Visual regression tests

```bash
idp scaffold test-suite --name my-visual --type visual --service my-svc \
  --threshold 0.1
```

| Flag | Default | Description |
|------|---------|-------------|
| `--threshold` | `0.2` | Max pixel diff ratio (0.0–1.0) |

---

#### `accessibility` — WCAG accessibility audit

```bash
idp scaffold test-suite --name my-a11y --type accessibility --service my-svc \
  --wcag wcag21aa
```

| Flag | Default | Description |
|------|---------|-------------|
| `--wcag` | `wcag2aa` | WCAG level: `wcag2a` \| `wcag2aa` \| `wcag21aa` \| `wcag22aa` |

---

#### `cucumber` — BDD tests

```bash
idp scaffold test-suite --name my-bdd --type cucumber --service my-svc
```

No extra flags. Generates a Cucumber/Gherkin suite with step definitions.

---

#### `appium` — Mobile tests

```bash
idp scaffold test-suite --name my-mobile --type appium --service my-svc \
  --platform ios --appium-server http://localhost:4723
```

| Flag | Default | Description |
|------|---------|-------------|
| `--platform` | `android` | Mobile platform: `android` \| `ios` |
| `--appium-server` | `http://localhost:4723` | Appium server URL |

---

#### `chaos` — Chaos engineering (Chaos Mesh)

```bash
idp scaffold test-suite --name my-chaos --type chaos --service my-svc \
  --experiments pod-failure,network-latency --chaos-duration 2m
```

| Flag | Default | Description |
|------|---------|-------------|
| `--experiments` | `pod-failure,network-latency` | Comma-separated experiment types |
| `--chaos-duration` | `1m` | Experiment duration (e.g. `1m`, `5m`) |

---

#### `mutation` — Mutation testing (Stryker)

```bash
idp scaffold test-suite --name my-mutation --type mutation --service my-svc \
  --score 80 --test-runner jest
```

| Flag | Default | Description |
|------|---------|-------------|
| `--score` | `70` | Minimum mutation score percentage |
| `--test-runner` | `jest` | Stryker test runner: `jest` \| `mocha` \| `jasmine` |

---

#### `testcontainers` — Integration tests with real containers

```bash
idp scaffold test-suite --name my-integration --type testcontainers --service my-svc \
  --containers postgres,redis
```

| Flag | Default | Description |
|------|---------|-------------|
| `--containers` | `postgres` | Comma-separated container images |

---

### Local output (`--local`)

All test suite types produce the same structure:

```
test-suites/<name>/
├── catalog-info.yaml
├── mkdocs.yml
└── <type-specific test files>
```

---

## Backstage API mode vs local mode

| | Backstage API mode | Local mode (`--local`) |
|---|---|---|
| **Trigger** | Backstage reachable at `--backstage-url` | Backstage unreachable, or `--local` flag set |
| **What happens** | Scaffolder API called → GitHub repo created, catalog entry registered, GitOps PR opened, TechDocs generated | Files written directly into this repo under `services/` or `test-suites/` |
| **Requires** | Running Backstage + valid auth token | Nothing (fully offline) |
| **Best for** | Production golden-path workflow | Local dev, CI pre-Backstage, quick iteration |

---

## Self-hosted GitHub Actions runner (optional)

Wire a self-hosted runner so CI/CD runs inside your local Kind cluster:

```bash
./scripts/setup-runner.sh --repo <service-name>
```
