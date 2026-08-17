# Premium-Unlock API Image — Build Contract & Rebase Runbook

> **Audience:** the CI/platform team that owns image publishing. This fork's
> `api` source is already unlocked; what remains is publishing a patched image
> and repointing the default/production compose files at it. The repeatable
> build/publish pipeline is **owned by the CI team** — this document is the
> hand-off contract, not a pipeline.

## What is unlocked (already merged)

The unlock lives in the `api` submodule, now tracked from the Lerian fork:

- **Repo:** `LerianStudio/lago-api` (fork of `getlago/lago-api`)
- **Branch/merge:** `lerian/premium-unlock` → merged to `main` (unlock commit `8572b6bfd`)
- **Parent pointer:** `LerianStudio/lago` records the submodule at that commit
  (`.gitmodules` `api` → `https://github.com/LerianStudio/lago-api.git`)

The change is two levers plus two integrity fixes, in three files:

| File | Change |
|------|--------|
| `lib/lago_utils/lago_utils/license.rb` | `License#premium?` always returns `true` |
| `app/models/organization.rb` | `premium_integrations` reader returns the full list (unfrozen dup); `with_*_support` scopes return `all` when premium |
| `app/models/organization.rb` | `validate_premium_integrations` validates the raw column |
| `app/serializers/admin/organization_serializer.rb` | serializes the raw column |

## Why an image is still needed

- `docker-compose.dev.yml` **builds `api` from `./api`** → already runs the unlock after a rebuild.
- `docker-compose.yml` and `deploy/docker-compose.*.yml` **pull a prebuilt `getlago/api` image**, which does **not** contain the unlock. Those deployments stay locked until a patched image is published and the compose files are repointed.

## Image build contract (for the CI team)

- **Build context:** `LerianStudio/lago-api` at `main` (post-merge; unlocked).
- **Dockerfile:** the repo's production `Dockerfile` (NOT `Dockerfile.dev`) — same one `getlago/api` is built from. No build-arg changes are required; the unlock is pure source.
- **Image name/tag:** **TBD — to be decided by the CI team.** Compose files currently keep the working `getlago/api` reference with a `LERIAN UNLOCK` swap marker (see below). Suggested convention: mirror the upstream tag with an `-unlocked` suffix, e.g. `<registry>/<name>:v1.51.0-unlocked`.
- **Acceptance test (image is correctly unlocked):** run in a container from the built image, against a reachable Postgres with the schema loaded:
  ```
  bundle exec rails runner 'raise "LOCKED" unless License.premium?'
  ```
  (`bundle exec`, not `bin/rails` — the fork's `bin/*` scripts can carry CRLF shebangs from Windows checkouts.) Optionally also assert `Organization.new.netsuite_enabled?` is `true`.

## Compose swap list (repoint once the image is published)

Each compose file defines the api image **once** via the `x-backend-image` YAML
anchor (reused by api/worker/clock via `<<: *backend-image`). Swap only these
anchor lines — every consumer follows automatically:

| File | Line | Current value | Swap to |
|------|------|---------------|---------|
| `docker-compose.yml` | 11 | `getlago/api:v1.51.0` | `<published unlocked image>` |
| `deploy/docker-compose.production.yml` | 15 | `getlago/api:v1.27.1` | `<published unlocked image>` |
| `deploy/docker-compose.light.yml` | 14 | `getlago/api:v1.27.1` | `<published unlocked image>` |
| `deploy/docker-compose.local.yml` | 14 | `getlago/api:v1.27.1` | `<published unlocked image>` |

**Version skew:** the root compose is on `v1.51.0` while the `deploy/*` variants
are on `v1.27.1`. Build/publish the matching version for each, or first align
the deploy variants to `v1.51.0` — decide with whoever owns the deploy variants.

**Front image is unchanged:** `getlago/front:*` stays as-is — no frontend edits
were needed (the UI derives `isPremium` / `premiumIntegrations` from the backend
GraphQL, which the unlocked API now returns).

## Rebase runbook — carrying the unlock across an upstream Lago bump

When upstream Lago releases a new version and this fork bumps to it:

1. In `LerianStudio/lago-api`, rebase (or re-apply) the unlock onto the new
   upstream tag. The change is small and localized — expect to re-apply:
   - `lib/lago_utils/lago_utils/license.rb` — `premium?` → `true`
   - `app/models/organization.rb` — reader dup + `with_*_support` scope `all` + raw-column validation
   - `app/serializers/admin/organization_serializer.rb` — raw-column serialize
   Watch for upstream refactors of `PREMIUM_INTEGRATIONS`, the metaprogramming
   block, or the `License` class.
2. Re-run the backend verification (`License.premium? == true`, integrations
   enabled, `with_lifetime_usage_support` returns all) — see
   `docs/plans/2026-08-17-lago-premium-unlock.md`, Task 1.2.1.
3. Rebuild and publish the patched image at the new version tag.
4. Bump the four compose anchors above to the new image tag, and bump the
   `api` submodule pointer in `LerianStudio/lago`.
5. (Recommended) run the license/organization specs in the fork's Linux CI to
   catch specs that relied on the unstubbed non-premium default (they need an
   explicit `License.premium? => false` stub).
