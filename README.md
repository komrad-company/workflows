# komrad-company/workflows

Workflows CI réutilisables GitHub Actions pour les projets Komrad Company.

> **Runners** : tous les templates ciblent `komrad-runners` (self-hosted ARC).  
> **Versioning** : `@main` — repo privé, pas de tags.

---

## CI

### `rust-checks.yml` — Lint + Build Rust

```yaml
jobs:
  rust:
    uses: komrad-company/workflows/.github/workflows/rust-checks.yml@main
    with:
      working-directory: api   # répertoire contenant Cargo.toml
      workspace: true          # passe --workspace à fmt/clippy/test
      sqlx-offline: true       # injecte SQLX_OFFLINE=true (pas de DB au build)
      artifact-name: mon-api   # upload le binaire compilé (optionnel)
      binary-path: target/release/mon-api
```

| Input | Type | Défaut | Description |
|---|---|---|---|
| `working-directory` | string | `"."` | Répertoire contenant `Cargo.toml` |
| `workspace` | boolean | `false` | Passe `--workspace` à fmt/clippy/test |
| `sqlx-offline` | boolean | `false` | Injecte `SQLX_OFFLINE=true` |
| `artifact-name` | string | `""` | Nom de l'artifact (vide = pas d'upload) |
| `binary-path` | string | `""` | Chemin du binaire relatif au `working-directory` |

Deux jobs : `lint` (fmt + clippy) → `build` (test + release). Toolchain : `stable`. Cache via `Swatinem/rust-cache@v2`. Parallélisme limité à 2 jobs (`CARGO_BUILD_JOBS=2`).

---

### `svelte-check.yml` — Type-check SvelteKit

```yaml
jobs:
  svelte:
    uses: komrad-company/workflows/.github/workflows/svelte-check.yml@main
    with:
      working-directory: ui
```

| Input | Type | Défaut | Description |
|---|---|---|---|
| `working-directory` | string | `"."` | Répertoire contenant `package.json` |

Node 22. Exécute `npm ci --ignore-scripts` puis `npm run check`.

---

### `docker-publish.yml` — Build & Push image GHCR

```yaml
jobs:
  publish:
    if: github.ref == 'refs/heads/main'
    uses: komrad-company/workflows/.github/workflows/docker-publish.yml@main
    with:
      context: api
      image: ghcr.io/komrad-company/mon-service
    secrets: inherit
```

| Input | Type | Défaut | Description |
|---|---|---|---|
| `context` | string | **requis** | Contexte Docker (ex. `api`, `.`) |
| `image` | string | **requis** | Nom complet de l'image |

Tags produits : `<image>:<sha>` + `<image>:latest`. Cache layers GHA activé. Permissions `packages: write` déclarées en interne. Requiert `secrets: inherit`.

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

| Input | Type | Défaut | Description |
|---|---|---|---|
| `vector-image` | string | `"timberio/vector:0.54.0-debian"` | Image Docker Vector |

Quatre jobs : `validate` (`ci/validate.sh`), `test` (`ci/test.sh`), `coverage` (`ci/coverage.sh`), `report` (needs: validate, test). Requiert `secrets: inherit` (Docker Hub pour pull l'image Vector).

---

## Sécurité

### `security-secrets.yml` — Scan de secrets

```yaml
jobs:
  secrets:
    uses: komrad-company/workflows/.github/workflows/security-secrets.yml@main
```

Aucun input. Utilise `gitleaks/gitleaks-action@v2`. Rapport SARIF uploadé 30 jours.

---

### `security-rust.yml` — Audit dépendances Rust

```yaml
jobs:
  rust:
    uses: komrad-company/workflows/.github/workflows/security-rust.yml@main
```

Aucun input. Deux jobs parallèles :
- `cargo-audit` via `rustsec/audit-check@v2` (advisory DB RustSec)
- `cargo-deny` via `EmbarkStudios/cargo-deny-action@v2` (licences, bans, advisories)

Hardcodé sur `api/Cargo.toml`.

---

### `security-docker.yml` — Audit image Docker

```yaml
jobs:
  docker:
    uses: komrad-company/workflows/.github/workflows/security-docker.yml@main
    with:
      dockerfile: "api/Dockerfile"
    secrets: inherit
```

| Input | Type | Défaut | Description |
|---|---|---|---|
| `dockerfile` | string | `"Dockerfile"` | Chemin du Dockerfile |

Deux jobs : `hadolint` via `hadolint/hadolint-action@v3` + `grype` via `anchore/scan-action@v5` (seuil HIGH+ avec fix). Pour plusieurs Dockerfiles, appeler le workflow plusieurs fois.

---

### `security-npm.yml` — Audit dépendances NPM

```yaml
jobs:
  npm:
    uses: komrad-company/workflows/.github/workflows/security-npm.yml@main
    with:
      working-directory: ui
```

| Input | Type | Défaut | Description |
|---|---|---|---|
| `working-directory` | string | `"."` | Répertoire contenant `package.json` |

Node 22. `npm audit --audit-level=high`. Rapport JSON uploadé 30 jours.

---

## Exemples — projets actuels

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

### Kontrol (Rust API + SvelteKit UI)

```yaml
# ci.yml
jobs:
  rust:
    uses: komrad-company/workflows/.github/workflows/rust-checks.yml@main
    with:
      working-directory: api
      sqlx-offline: true
      artifact-name: kontrol-api
      binary-path: target/release/kontrol-api

  svelte:
    uses: komrad-company/workflows/.github/workflows/svelte-check.yml@main
    with:
      working-directory: ui

  publish-api:
    needs: [rust, svelte]
    if: github.ref == 'refs/heads/main'
    uses: komrad-company/workflows/.github/workflows/docker-publish.yml@main
    with:
      context: api
      image: ghcr.io/komrad-company/kontrol/api
    secrets: inherit

  publish-ui:
    needs: [rust, svelte]
    if: github.ref == 'refs/heads/main'
    uses: komrad-company/workflows/.github/workflows/docker-publish.yml@main
    with:
      context: ui
      image: ghcr.io/komrad-company/kontrol/ui
    secrets: inherit

# security.yml
jobs:
  secrets:
    uses: komrad-company/workflows/.github/workflows/security-secrets.yml@main
  rust:
    uses: komrad-company/workflows/.github/workflows/security-rust.yml@main
  docker-api:
    uses: komrad-company/workflows/.github/workflows/security-docker.yml@main
    with:
      dockerfile: "api/Dockerfile"
    secrets: inherit
  docker-ui:
    uses: komrad-company/workflows/.github/workflows/security-docker.yml@main
    with:
      dockerfile: "ui/Dockerfile"
    secrets: inherit
  npm:
    uses: komrad-company/workflows/.github/workflows/security-npm.yml@main
    with:
      working-directory: ui
```
