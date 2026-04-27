# komrad-company/workflows

Templates CI réutilisables GitHub Actions pour les projets Komrad Company.

## Utilisation rapide

```yaml
jobs:
  rust:
    uses: komrad-company/workflows/.github/workflows/rust-checks.yml@main
    with:
      rust-version: "1.94"
      working-directory: api
      workspace: true
```

> **Runners** : tous les templates ciblent `komrad-runners` (self-hosted).  
> **Versioning** : `@main` — repo privé, pas de tags.  
> **Secrets** : toujours passer `secrets: inherit` sur les jobs qui appellent `docker-publish.yml`.

---

## Templates disponibles

### `rust-checks.yml` — Lint + Build Rust

| Input | Type | Défaut | Description |
|---|---|---|---|
| `rust-version` | string | `"stable"` | Version toolchain (ex. `"1.94"`) |
| `working-directory` | string | `"."` | Répertoire contenant `Cargo.toml` |
| `workspace` | boolean | `false` | Passe `--workspace` à fmt/clippy/test |
| `sqlx-offline` | boolean | `false` | Injecte `SQLX_OFFLINE=true` (pas de DB au build) |
| `run-tests` | boolean | `true` | Active `cargo test` |
| `run-build` | boolean | `true` | Active `cargo build --release` |
| `artifact-name` | string | `""` | Nom de l'artifact uploadé (vide = pas d'upload) |
| `binary-path` | string | `""` | Chemin du binaire relatif au `working-directory` |

Deux jobs internes : `lint` (fmt + clippy, parallèles) et `build` (test + release, needs: lint).  
Cache via `Swatinem/rust-cache@v2`.

---

### `docker-publish.yml` — Build & Push image GHCR

| Input | Type | Défaut | Description |
|---|---|---|---|
| `context` | string | **requis** | Contexte Docker (ex. `api`, `.`) |
| `image` | string | **requis** | Nom complet image (ex. `ghcr.io/komrad-company/kolektor`) |
| `gha-cache` | boolean | `true` | Active le cache layers Docker via GHA |

Tags produits : `<image>:<sha>` + `<image>:latest`.  
Permissions `packages: write` déclarées en interne.  
Requiert `secrets: inherit` côté appelant (GITHUB_TOKEN).

---

### `svelte-check.yml` — Type-check SvelteKit

| Input | Type | Défaut | Description |
|---|---|---|---|
| `node-version` | string | `"22"` | Version Node.js |
| `working-directory` | string | `"."` | Répertoire contenant `package.json` |
| `lockfile-path` | string | `"package-lock.json"` | Chemin du lockfile pour le cache npm |

Exécute `npm ci --ignore-scripts` puis `npm run check`.

---

### `vector-checks.yml` — Validation & tests Vector.dev

| Input | Type | Défaut | Description |
|---|---|---|---|
| `vector-image` | string | `"timberio/vector:0.54.0-debian"` | Image Docker Vector |
| `validate-script` | string | `"ci/validate.sh"` | Script de validation des configs |
| `test-script` | string | `"ci/test.sh"` | Script de tests |
| `coverage-script` | string | `"ci/coverage.sh"` | Script de couverture |

Quatre jobs : `validate`, `test`, `coverage` (parallèles) + `report` (needs: validate, test).  
Artifacts : `validate-results`, `test-results`, `ci-report`.

---

### `security-secrets.yml` — Scan de secrets (universel)

Aucun input. Scan gitleaks v8.21.2 sur tout le repo.  
Artifacts SARIF + JSON uploadés 30 jours.  
**Utilisable par tous les repos** — aucune toolchain requise.

---

### `security-rust.yml` — Audit dépendances Rust

| Input | Type | Défaut | Description |
|---|---|---|---|
| `rust-version` | string | `"stable"` | Version toolchain Rust |
| `working-directory` | string | `"."` | Répertoire contenant `Cargo.toml` |

Deux jobs parallèles : `cargo-audit` (RustSec advisory DB) + `cargo-deny` (licences, bans, advisories).  
**Rust uniquement** — pour les Dockerfiles, utiliser `security-docker.yml`.

---

### `security-docker.yml` — Audit images Docker

| Input | Type | Défaut | Description |
|---|---|---|---|
| `dockerfiles` | string | `"Dockerfile"` | Chemins séparés par espaces (ex. `"api/Dockerfile ui/Dockerfile"`) |

Deux jobs : `hadolint` v2.12.0 (lint best practices, seuil error) + `grype` latest (vulnérabilités image, seuil HIGH+ avec fix disponible).  
**Langage-agnostique** — fonctionne sur tout repo ayant un Dockerfile.  
Artifacts : `hadolint-report` (SARIF) + `grype-report` (JSON).

---

### `security-npm.yml` — Audit dépendances NPM

| Input | Type | Défaut | Description |
|---|---|---|---|
| `node-version` | string | `"22"` | Version Node.js |
| `working-directory` | string | `"."` | Répertoire contenant `package.json` |
| `lockfile-path` | string | `"package-lock.json"` | Chemin du lockfile pour le cache |

Job `npm-audit` : `npm ci --ignore-scripts` + `npm audit --audit-level=high` (gating) + JSON complet uploadé.  
**JS/TS uniquement** — aucun Rust requis.

---

## Exemples complets

### Projet Rust simple (lint uniquement)

```yaml
# .github/workflows/ci.yml
jobs:
  lint:
    uses: komrad-company/workflows/.github/workflows/rust-checks.yml@main
    with:
      run-tests: false
      run-build: false

# .github/workflows/security.yml
jobs:
  secrets:
    uses: komrad-company/workflows/.github/workflows/security-secrets.yml@main
  rust:
    uses: komrad-company/workflows/.github/workflows/security-rust.yml@main
```

### Service Rust + Docker (Kolektor)

```yaml
# .github/workflows/ci.yml
jobs:
  vector:
    uses: komrad-company/workflows/.github/workflows/vector-checks.yml@main
    with:
      vector-image: timberio/vector:0.54.0-debian

  rust:
    uses: komrad-company/workflows/.github/workflows/rust-checks.yml@main
    with:
      rust-version: "1.94"
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

# .github/workflows/security.yml
jobs:
  secrets:
    uses: komrad-company/workflows/.github/workflows/security-secrets.yml@main
  rust:
    uses: komrad-company/workflows/.github/workflows/security-rust.yml@main
    with:
      rust-version: "1.94"
      working-directory: api
  docker:
    uses: komrad-company/workflows/.github/workflows/security-docker.yml@main
    with:
      dockerfiles: "Dockerfile"
```

### Fullstack Rust + SvelteKit (Kontrol)

```yaml
# .github/workflows/ci.yml
jobs:
  rust:
    uses: komrad-company/workflows/.github/workflows/rust-checks.yml@main
    with:
      rust-version: "1.95"
      working-directory: api
      sqlx-offline: true
      artifact-name: kontrol-api
      binary-path: target/release/kontrol-api

  svelte:
    uses: komrad-company/workflows/.github/workflows/svelte-check.yml@main
    with:
      working-directory: ui
      lockfile-path: ui/package-lock.json

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

# .github/workflows/security.yml
jobs:
  secrets:
    uses: komrad-company/workflows/.github/workflows/security-secrets.yml@main
  rust:
    uses: komrad-company/workflows/.github/workflows/security-rust.yml@main
    with:
      rust-version: "1.95"
      working-directory: api
  docker:
    uses: komrad-company/workflows/.github/workflows/security-docker.yml@main
    with:
      dockerfiles: "api/Dockerfile ui/Dockerfile"
  npm:
    uses: komrad-company/workflows/.github/workflows/security-npm.yml@main
    with:
      working-directory: ui
      lockfile-path: ui/package-lock.json
```
