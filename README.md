# komrad-company/workflows

Reusable GitHub Actions CI workflows for Komrad Company projects.

> **Runners**: all templates target `komrad-runners` (self-hosted ARC).  
> **Versioning**: `@main` — private repo, no tags.

---

## CI

### `rust-checks.yml` — Lint + Build Rust

```yaml
jobs:
  rust:
    uses: komrad-company/workflows/.github/workflows/rust-checks.yml@main
    with:
      working-directory: api   # directory containing Cargo.toml
      workspace: true          # passes --workspace to fmt/clippy/test
      sqlx-offline: true       # injects SQLX_OFFLINE=true (no DB at build time)
      artifact-name: my-api    # upload the compiled binary (optional)
      binary-path: target/release/my-api
```

| Input | Type | Default | Description |
|---|---|---|---|
| `working-directory` | string | `"."` | Directory containing `Cargo.toml` |
| `workspace` | boolean | `false` | Passes `--workspace` to fmt/clippy/test |
| `sqlx-offline` | boolean | `false` | Injects `SQLX_OFFLINE=true` |
| `artifact-name` | string | `""` | Artifact name (empty = no upload) |
| `binary-path` | string | `""` | Binary path relative to `working-directory` |

Two jobs: `lint` (fmt + clippy) → `build` (test + release). Toolchain: `stable`. Cache via `Swatinem/rust-cache@v2`. Parallelism limited to 2 jobs (`CARGO_BUILD_JOBS=2`).

---

### `svelte-check.yml` — Type-check SvelteKit

```yaml
jobs:
  svelte:
    uses: komrad-company/workflows/.github/workflows/svelte-check.yml@main
    with:
      working-directory: ui
```

| Input | Type | Default | Description |
|---|---|---|---|
| `working-directory` | string | `"."` | Directory containing `package.json` |

Node 22. Runs `npm ci --ignore-scripts` then `npm run check`.

---

### `docker-publish.yml` — Build & Push GHCR image

```yaml
jobs:
  publish:
    if: github.ref == 'refs/heads/main'
    uses: komrad-company/workflows/.github/workflows/docker-publish.yml@main
    with:
      context: api
      image: ghcr.io/komrad-company/my-service
    secrets: inherit
```

| Input | Type | Default | Description |
|---|---|---|---|
| `context` | string | **required** | Docker context (e.g. `api`, `.`) |
| `image` | string | **required** | Full image name |

Tags produced: `<image>:<sha>` + `<image>:latest`. GHA layer cache enabled. `packages: write` permission declared internally. Requires `secrets: inherit`.

---

### `vector-checks.yml` — Validation & tests Vector.dev

```yaml
jobs:
  vector:
    uses: komrad-company/workflows/.github/workflows/vector-checks.yml@main
    with:
      vector-image: timberio/vector:0.54.0-debian
    secrets: inherit
```

| Input | Type | Default | Description |
|---|---|---|---|
| `vector-image` | string | `"timberio/vector:0.54.0-debian"` | Vector Docker image |

Four jobs: `validate` (`ci/validate.sh`), `test` (`ci/test.sh`), `coverage` (`ci/coverage.sh`), `report` (needs: validate, test). Requires `secrets: inherit` (Docker Hub to pull the Vector image).

---

## Security

### `security-secrets.yml` — Secret scanning

```yaml
jobs:
  secrets:
    uses: komrad-company/workflows/.github/workflows/security-secrets.yml@main
```

No inputs. Downloads `gitleaks` v8.30.1 then runs
`gitleaks detect --source . --verbose` with `fetch-depth: 0`.

---

### `security-rust.yml` — Rust dependency audit

```yaml
jobs:
  rust:
    uses: komrad-company/workflows/.github/workflows/security-rust.yml@main
    with:
      manifest-path: api/Cargo.toml
```

| Input | Type | Default | Description |
|---|---|---|---|
| `manifest-path` | string | `"api/Cargo.toml"` | Path to the `Cargo.toml` analysed by `cargo-deny` |

Two parallel jobs:
- `cargo-audit` via `rustsec/audit-check@v2` (RustSec advisory DB)
- `cargo-deny` via `EmbarkStudios/cargo-deny-action@v2` (licenses, bans, advisories)

The default remains `api/Cargo.toml` for legacy monorepos. For a root Rust repo,
pass `manifest-path: Cargo.toml`.

---

### `security-docker.yml` — Docker image audit

```yaml
jobs:
  docker:
    uses: komrad-company/workflows/.github/workflows/security-docker.yml@main
    with:
      dockerfile: "api/Dockerfile"
    secrets: inherit
```

| Input | Type | Default | Description |
|---|---|---|---|
| `dockerfile` | string | `"Dockerfile"` | Path to the Dockerfile |

Two jobs: `hadolint` via `hadolint/hadolint-action@v3.3.0` + `grype` via `anchore/scan-action@v5` (HIGH+ threshold with fix). For multiple Dockerfiles, call the workflow multiple times.

---

### `security-npm.yml` — NPM dependency audit

```yaml
jobs:
  npm:
    uses: komrad-company/workflows/.github/workflows/security-npm.yml@main
    with:
      working-directory: ui
```

| Input | Type | Default | Description |
|---|---|---|---|
| `working-directory` | string | `"."` | Directory containing `package.json` |

Node 22. `npm audit --audit-level=high`. JSON report uploaded for 30 days.

---

## Examples — current projects

### Kolektor (Vector + Rust + Docker)

```yaml
# ci.yml
jobs:
  vector:
    uses: komrad-company/workflows/.github/workflows/vector-checks.yml@main
    with:
      vector-image: timberio/vector:0.54.0-debian
    secrets: inherit

  rust:
    uses: komrad-company/workflows/.github/workflows/rust-checks.yml@main
    with:
      working-directory: api
      workspace: true

  publish:
    needs: [vector, rust]
    if: github.ref == 'refs/heads/main'
    uses: komrad-company/workflows/.github/workflows/docker-publish.yml@main
    with:
      context: .
      image: ghcr.io/komrad-company/kolektor
    secrets: inherit

# security.yml
jobs:
  secrets:
    uses: komrad-company/workflows/.github/workflows/security-secrets.yml@main
  rust:
    uses: komrad-company/workflows/.github/workflows/security-rust.yml@main
  docker:
    uses: komrad-company/workflows/.github/workflows/security-docker.yml@main
    secrets: inherit
```

### Kontrol-api (Rust API)

```yaml
# ci.yml
jobs:
  rust:
    uses: komrad-company/workflows/.github/workflows/rust-checks.yml@main
    with:
      sqlx-offline: true
      artifact-name: kontrol-api
      binary-path: target/release/kontrol-api

  publish:
    needs: [rust]
    if: github.ref == 'refs/heads/main'
    uses: komrad-company/workflows/.github/workflows/docker-publish.yml@main
    with:
      context: .
      image: ghcr.io/komrad-company/kontrol-api
    secrets: inherit

# security.yml
jobs:
  secrets:
    uses: komrad-company/workflows/.github/workflows/security-secrets.yml@main
  rust:
    uses: komrad-company/workflows/.github/workflows/security-rust.yml@main
    with:
      manifest-path: Cargo.toml
  docker:
    uses: komrad-company/workflows/.github/workflows/security-docker.yml@main
    secrets: inherit
```

### Kontrol-ui (SvelteKit UI)

```yaml
# ci.yml
jobs:
  svelte:
    uses: komrad-company/workflows/.github/workflows/svelte-check.yml@main

  publish:
    needs: [svelte]
    if: github.ref == 'refs/heads/main'
    uses: komrad-company/workflows/.github/workflows/docker-publish.yml@main
    with:
      context: .
      image: ghcr.io/komrad-company/kontrol-ui
    secrets: inherit

# security.yml
jobs:
  secrets:
    uses: komrad-company/workflows/.github/workflows/security-secrets.yml@main
  npm:
    uses: komrad-company/workflows/.github/workflows/security-npm.yml@main
  docker:
    uses: komrad-company/workflows/.github/workflows/security-docker.yml@main
    secrets: inherit
```
