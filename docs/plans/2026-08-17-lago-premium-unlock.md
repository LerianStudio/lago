# Lago Premium Unlock Implementation Plan

> **For implementers:** Use ring:executing-plans (rolling-phase: elaborate the
> current phase against the real code, execute its tasks in review-checkpointed
> batches, then elaborate the next phase — repeat),
> ring:dispatching-workflows to run each phase as a reviewed multi-agent
> workflow (review + contrarian baked in), or ring:running-dev-cycle for the
> full subagent-orchestrated workflow.
> This document is the living source of truth — task elaboration for later
> phases is written back into it during execution.

**Goal:** Unlock all Lago Premium capabilities in this fork by removing the license gate at its two backend chokepoints, then persist the change durably in a Lerian fork of `lago-api` and ship it to both the dev and production run modes.

**Architecture:** Lago gates premium behind exactly two backend switches: a global `License#premium?` boolean (drives ~180 direct gates) and a per-organization `premium_integrations` array (drives the metaprogrammed `<integration>_enabled?` predicates, the `with_*_support` SQL scopes, and the GraphQL field the frontend reads). We flip both **in the backend only**: `premium?` returns `true`, and every organization behaves as if it holds all premium integrations. The frontend (`lago-front`) needs **zero changes** — it derives `isPremium` and `hasOrganizationPremiumAddon()` entirely from these backend GraphQL fields. Persistence follows the fork model: patched source lives on a branch of `LerianStudio/lago-api`, the parent repo's `api` submodule repoints to it, and the production image is rebuilt from it.

**Tech Stack:** Ruby on Rails (lago-api), GraphQL, PostgreSQL, Docker Compose, git submodules. Frontend is TypeScript/React/Apollo (lago-front) — untouched.

## Phase Overview

| Phase | Milestone | Epics | Status |
|-------|-----------|-------|--------|
| 1 | Premium fully unlocked and verified in the dev stack (`docker-compose.dev.yml` builds from `./api`), persisted on a Lerian fork with the parent `api` submodule repointed | 1.1, 1.2, 1.3 | Detailed |
| 2 | Build contract + compose swap markers + rebase runbook ready for the CI team to publish the patched image (CI pipeline out of scope) | 2.1, 2.2, 2.3 | Complete |

---

## Context: the lock system (verified map)

Two independent switches, both in `lago-api`:

1. **Global flag** — `LagoUtils::License#premium?` at `api/lib/lago_utils/lago_utils/license.rb:19-21` returns the in-memory `@premium` boolean. It is set once at boot by `#verify`, which returns early (leaving `@premium = false`) unless `ENV["LAGO_LICENSE"]` is present and a remote `GET {license_url}/verify/{key}` responds `{"valid": true}`. The global singleton is built in `api/config/initializers/license.rb`. This flag guards ~180 direct `License.premium?` gates across services, GraphQL resolvers, jobs, and controllers (via the `PremiumFeatureOnly` concern). It is also surfaced to the frontend as `User.premium` (`api/app/graphql/types/user_type.rb:25-27`).

2. **Per-organization integrations** — the `organizations.premium_integrations` array column, whose allowed values are `Organization::PREMIUM_INTEGRATIONS` (`api/app/models/organization.rb:128-158`). The metaprogramming block at `api/app/models/organization.rb:194-200` derives, per integration: a `with_#{name}_support` SQL scope (`WHERE ? = ANY(premium_integrations)`) and an `#{name}_enabled?` predicate (`License.premium? && premium_integrations.include?(name)`). The GraphQL field `CurrentOrganization.premiumIntegrations` (`api/app/graphql/types/organizations/current_organization_type.rb:45`) returns this array directly; the frontend's `hasOrganizationPremiumAddon()` (`front/src/hooks/useOrganizationInfos.ts:98-99`) checks membership in it.

**Why both switches, and why the reader + scope both matter:** the predicates (`#{name}_enabled?`) and the GraphQL field read the model attribute, but the two background jobs that select orgs by integration — `api/app/jobs/clock/refresh_lifetime_usages_job.rb:11-12` and `api/app/services/daily_usages/compute_all_service.rb:16` — use the `with_*_support` **SQL scopes**, which query the raw column and bypass any Ruby-level attribute override. So a reader-only override would silently leave lifetime-usage and daily-usage recomputation processing zero organizations. Lever 2 therefore overrides **both** the reader (for predicates + GraphQL + frontend) and the scopes (for the jobs).

**Frontend needs no changes:** `front/src/hooks/useCurrentUser.ts:105` sets `isPremium` from `currentUser.premium`, and `useOrganizationInfos.ts:90,98-99` reads `premiumIntegrations`. Both come from the backend GraphQL above. Fixing the backend flows through automatically.

---

## Phase 1 — Unlock and verify in dev, persisted on the fork

### Epic 1.1: Two-lever backend unlock [map:#4094]

**Goal:** `License.premium?` is `true` and every organization reports all premium integrations enabled, at the model layer, in the `api/` submodule working tree.
**Scope:** `api/lib/lago_utils/lago_utils/license.rb`, `api/app/models/organization.rb`
**Dependencies:** none
**Done when:** in a Rails console, `License.premium?` is `true`; `Organization.first.premium_integrations` returns the full `PREMIUM_INTEGRATIONS` list; `Organization.first.netsuite_enabled?` (and any other `*_enabled?`) is `true`; `Organization.with_lifetime_usage_support.count == Organization.count`.
**Status:** Pending

> **Execution note (2026-08-17):** Full-grant semantics confirmed by the user — every org gets every premium integration truly enabled, including passive side effects (`from_email`, `issue_receipts`, `security_logs`, daily-usage jobs across all orgs). Code review surfaced and we fixed three correctness issues beyond the two levers: (1) reader returns `PREMIUM_INTEGRATIONS.dup` to avoid `FrozenError` on `<<`; (2) `validate_premium_integrations` reads the raw column via `read_attribute` so invalid assigned values are still rejected; (3) `Admin::OrganizationSerializer` serializes the raw column (`model[:premium_integrations]`) so operators still see the purchased subset and a Console round-trip doesn't overwrite it. The passive side effects are accepted, not defects.

#### Task 1.1.1: Force the global license flag to premium

- [x] Done

**Context:** `LagoUtils::License#premium?` at `api/lib/lago_utils/lago_utils/license.rb:19-21` returns the private `@premium` attribute, which defaults to `false` (`license.rb:7`) and only becomes `true` when a valid `LAGO_LICENSE` key validates against the remote license server. This is the single boolean behind ~180 backend gates and the frontend `isPremium`.

**Implementation vision:** Make `#premium?` return `true` unconditionally — the clearest, most self-documenting unlock point. Change the method body from `premium` to `true`, and add a one-line comment marking it as the Lerian unlock so future upstream rebases surface it. Leave `#verify`, `@premium`, the initializer, and the `url`/`attr_reader` intact so nothing else about the boot sequence changes (no dead-code removal, minimal diff for clean rebases). Edge case — **test suite**: specs that explicitly `allow(License).to receive(:premium?).and_return(false)` still work, because the stub replaces the method; only specs relying on the *unstubbed default being false* change behavior (triaged in Task 1.2.3). Do not alter `#verify`: leaving the remote call path in place keeps the diff surgical and avoids touching `LagoHttpClient`.

**Files:**
- Modify: `api/lib/lago_utils/lago_utils/license.rb:19-21`

**Verification:** In the dev container, `lago exec api bin/rails runner 'puts License.premium?'` prints `true` (run after Task 1.2.1 rebuilds the image; before that, confirm by reading the diff).

**Done when:** `#premium?` returns `true` regardless of `LAGO_LICENSE` or the remote server, with the rest of the class unchanged.

#### Task 1.1.2: Auto-enable all premium integrations for every organization

- [x] Done

**Context:** The metaprogramming block at `api/app/models/organization.rb:194-200` defines, per premium integration, a `with_#{name}_support` scope (raw SQL on the `premium_integrations` column) and an `#{name}_enabled?` predicate (`License.premium? && premium_integrations.include?(name)`). The GraphQL field `CurrentOrganization.premiumIntegrations` and the frontend read the `premium_integrations` attribute; the two jobs named in the Context section read the SQL scopes. With Lever 1 done, `License.premium?` is `true`, but the array column is empty by default (`'{}'`), so predicates, GraphQL, and scopes all still report "not enabled." We must make every org report the full integration set without requiring per-org DB writes.

**Implementation vision:** Keep the DB column as the stored source of truth but override its *read path* and the *scopes* to present "all enabled" whenever `License.premium?`. Two edits in `organization.rb`:

1. Override the `premium_integrations` reader to return the full frozen `PREMIUM_INTEGRATIONS` list when `License.premium?`, else fall through to the stored value via `super`. This makes the GraphQL field, the frontend `hasOrganizationPremiumAddon()`, and every `#{name}_enabled?` predicate (which calls `premium_integrations.include?`) resolve to enabled — **without editing line 198's predicate**, which keeps working unchanged through the overridden reader.
2. Change the metaprogrammed scope so `with_#{name}_support` returns `all` when `License.premium?`, else the original `where(...)`. This is what makes `refresh_lifetime_usages_job` and `daily_usages/compute_all_service` iterate every org instead of none.

Both stay inside/next to the existing `PREMIUM_INTEGRATIONS.each` block so the whole lever lives in one place. The `validate_premium_integrations` validation (`organization.rb:184`, `:301-305`) still passes because every value in `PREMIUM_INTEGRATIONS` is valid by definition. The admin write path (`Admin::Organizations::UpdateService`) may still assign a subset to the column, but the reader override intentionally supersedes it — for a fully-unlocked fork, "all on" is the desired behavior. Edge case — **scope evaluation timing**: the scope is a lambda, so `License.premium?` is evaluated at call time (correct), not at class-load time. Edge case — **`super` in the reader**: `super` invokes the ActiveRecord-generated attribute reader; do not reference `@premium_integrations` directly. This is the contract other epics depend on, pinned below.

```ruby
# api/app/models/organization.rb — replaces the block at 194-200
# Lerian premium unlock: when licensed, every org reports all premium integrations.
def premium_integrations
  License.premium? ? PREMIUM_INTEGRATIONS : super
end

PREMIUM_INTEGRATIONS.each do |premium_integration|
  scope "with_#{premium_integration}_support", lambda {
    License.premium? ? all : where("? = ANY(premium_integrations)", premium_integration)
  }

  define_method("#{premium_integration}_enabled?") do
    License.premium? && premium_integrations.include?(premium_integration)
  end
end
```

**Files:**
- Modify: `api/app/models/organization.rb:194-200` (plus the new reader method immediately above the block)

**Verification:** `lago exec api bin/rails runner 'o=Organization.first; puts o.premium_integrations.sort == Organization::PREMIUM_INTEGRATIONS.sort; puts o.netsuite_enabled?; puts Organization.with_lifetime_usage_support.count == Organization.count'` prints `true` three times (run after the dev rebuild in Task 1.2.1).

**Done when:** the reader returns the full list and every `with_*_support` scope returns all orgs while `License.premium?` is `true`; predicate at line 198 is unchanged and resolves `true` through the reader.

---

### Epic 1.2: Dev verification and license-dependent spec triage [map:#4095]

**Goal:** the unlock is proven end-to-end against a running dev stack, and the impact on the existing test suite is known and resolved.
**Scope:** `docker-compose.dev.yml` dev stack (built from `./api` and `./front`), `api/spec/`
**Dependencies:** Epic 1.1
**Done when:** the dev stack runs the patched image; GraphQL `currentUser.premium` is `true` and `organization.premiumIntegrations` lists all integrations; the frontend renders premium features without paywalls; the api spec suite is green (or every remaining failure is a documented, accepted consequence of the unlock).
**Status:** Pending

#### Task 1.2.1: Rebuild the dev stack and verify premium at the backend

- [x] Done

> **Verified (2026-08-17):** `api_dev` image built; Postgres schema loaded directly from `api/db/structure.sql` into the `db` container (bypassing the ClickHouse-coupled `db:schema:load` and CRLF-broken script shebangs). `rails runner` confirmed: `License.premium? => true`; `Organization#premium_integrations` returns the full 29 as an unfrozen dup; `netsuite_enabled?`/`security_logs_enabled? => true`; `with_lifetime_usage_support` emits `SELECT … FROM organizations` with **no WHERE** (scope returns all). **Local env caveats** (Windows checkout): submodule files carry CRLF, breaking `#!/usr/bin/env` shebangs in the Linux containers (use `bundle exec`, not `bin/*`); `redpandacreatetopics` exits 255 and aborts a full `docker compose up`; boot requires `LAGO_RSA_PRIVATE_KEY` (generate one for one-off runs). None affect the fork's Linux CI.

**Context:** `docker-compose.dev.yml:137-139` builds the `api` image from `./api` with `Dockerfile.dev`; the same `api_dev` image is reused by worker/clock services (`docker-compose.dev.yml:150,195,301`). Source edits from Epic 1.1 only take effect after a rebuild. The repo wraps compose in a `lago` CLI (per `api/AGENTS.md`, commands run as `lago exec api ...`).

**Implementation vision:** Rebuild the `api` image and restart the API + worker/clock services so the metaprogramming and license changes load. Then verify at two layers: (a) Rails console — `License.premium?`, `Organization#premium_integrations`, a representative `*_enabled?`, and a `with_*_support` scope count (the Verification commands from Tasks 1.1.1/1.1.2); (b) GraphQL — execute the `getCurrentUserInfos` and `getOrganizationInfos` queries (the exact operations the frontend uses, defined at `front/src/hooks/useCurrentUser.ts:15-40` and `front/src/hooks/useOrganizationInfos.ts:22-46`) against the running API and assert `currentUser.premium == true` and `organization.premiumIntegrations` contains all values. Named edge case — **stale worker image**: the clock/worker services share the `api_dev` image tag; rebuild once and recreate all of them so the background jobs also run patched code, otherwise console checks pass while jobs still run old code.

**Files:**
- No source changes (verification task). Commands run against `docker-compose.dev.yml`.

**Verification:** `docker compose -f docker-compose.dev.yml build api && docker compose -f docker-compose.dev.yml up -d api api-worker api-clock` (or the `lago` CLI equivalent) succeeds; the three Rails-runner one-liners from Epic 1.1 print `true`; a GraphQL call to `getCurrentUserInfos` returns `currentUser.premium: true`.

**Done when:** the running dev API reports premium and all integrations at both the console and GraphQL layers.

#### Task 1.2.2: Frontend smoke — confirm paywalls are gone

- [ ] Done

**Context:** The frontend derives everything from the backend GraphQL of Task 1.2.1 — no frontend edits are in scope. Premium gates manifest as: the `PremiumWarningDialog` upsell (`front/src/components/dialogs/PremiumWarningDialog.tsx`), the `PremiumFeature` inline banner (`front/src/components/premium/PremiumFeature.tsx`), `sparkles`-icon gated buttons, blurred analytics/wallet views (`front/src/pages/OldAnalytics.tsx`, `front/src/components/wallets/WalletTransactions.tsx`), and hidden tabs (activity logs, security logs, custom roles). This task confirms the backend change actually clears them.

**Implementation vision:** With the dev frontend running (`docker-compose.dev.yml:82-91`, `front_dev` built from `./front`), log in and spot-check a representative gate from each family so we cover both the global flag and the per-addon path: (1) global-flag gate — open a plan's **Activity Logs** tab (hidden when `!isPremium`) and confirm it renders; (2) per-addon gate — open **Analytics/Dashboards** (guarded by `AnalyticsDashboards` addon) and confirm graphs are not blurred and no demo banner shows; (3) an action gate — open an invoice and confirm the "Issue credit note" / "Record payment" buttons render without the `sparkles` premium icon and do not open `PremiumWarningDialog`. No code changes expected. If any gate still shows a paywall, the backend GraphQL response is the culprit — re-check Task 1.2.1, not the frontend.

**Files:**
- No source changes (verification task).

**Verification:** Manual UI walkthrough of the three gates above against the dev frontend; each renders its premium content with no paywall/blur/sparkles.

**Done when:** the three representative gates (global-flag tab, per-addon analytics, action button) all render unlocked with no frontend edits.

#### Task 1.2.3: Run and triage license-dependent specs

- [ ] Done

**Context:** Forcing `License.premium?` to `true` and overriding the org reader/scopes changes the *default* (unstubbed) premium behavior in the test environment, where `License.verify` is skipped and `@premium` would otherwise be `false` (`api/config/initializers/license.rb:7`). Specs that assert non-premium/forbidden behavior *without* stubbing `License` will now see premium and may fail. Specs that already `allow(License).to receive(:premium?)` are unaffected (the stub wins). Shared helpers exist at `api/spec/support/shared_examples/api_requirements.rb:10`.

**Implementation vision:** Run the license/premium-adjacent slices of the suite and triage failures into two buckets. First, the license unit spec and the organization model spec: `lago exec api bundle exec rspec spec/lib/lago_utils/license_spec.rb spec/models/organization_spec.rb`. Then the premium-gate integration points most likely to assert forbidden-by-default (analytics resolvers/services, `PremiumFeatureOnly`, subscription/plan override services). For each failure decide: **(a)** the spec relied on the unstubbed default being non-premium — update it to explicitly stub `License.premium?` to `false` for the "unlicensed" case (this restores the test's intent and keeps it meaningful under the unlock), or **(b)** the failure reflects genuinely desired new behavior (a feature now allowed) — update the expectation. Do **not** weaken the `license_spec.rb` premium?-returns behavior test into meaninglessness; instead assert the new contract (`premium?` is `true`) and keep the `verify` tests as-is since `verify` is unchanged. Record any intentionally-changed expectations in the epic notes. Follow `api/AGENTS.md` testing rules (narrow the run, `have_received` over `receive`, no `aggregate_failures` in new/edited tests). Edge case — **do not** globally stub `License.premium?` in `rails_helper`; keep changes local to the specs that break, so future upstream test changes rebase cleanly.

**Files:**
- Modify: `api/spec/lib/lago_utils/license_spec.rb` (assert the new `premium? == true` contract)
- Modify: license-dependent specs identified during the run (exact list produced at execution time; expected to include `spec/models/organization_spec.rb` and a subset of analytics/premium-gate specs)

**Verification:** The targeted rspec runs are green; a documented list of edited specs with the (a)/(b) classification for each accompanies the change.

**Done when:** the license/organization specs and the premium-gate slices pass, with every edit classified as intent-preserving (a) or intentional new behavior (b).

---

### Epic 1.3: Persist via Lerian fork + submodule repoint [map:#4096]

**Goal:** the patched `lago-api` source lives on a branch of `LerianStudio/lago-api`, and the parent repo's `api` submodule points at it so `git submodule update` reproduces the unlock.
**Scope:** `LerianStudio/lago-api` (new fork), parent repo `.gitmodules` + `api` submodule pointer
**Dependencies:** Epic 1.1 (the change to commit), Epic 1.2 (verified before persisting)
**Done when:** a fresh `git submodule update --init` in the parent repo checks out the patched, unlocked `lago-api`; `.gitmodules` `api` url points at the Lerian fork; the parent repo has a commit recording the new submodule SHA.
**Status:** Pending

#### Task 1.3.1: Fork lago-api to LerianStudio and push the unlock branch

- [x] Done

> **Done (2026-08-17):** Fork `LerianStudio/lago-api` created (of `getlago/lago-api`); unlock committed as `8572b6bfd` on branch `lerian/premium-unlock` (conventional message + Co-Authored-By), pushed to the fork via the `lerian` https remote. `core.autocrlf=true` in the submodule stored LF despite the CRLF working tree, so the commit is exactly the 3 intended files.

**Context:** The `api` submodule currently tracks `git@github.com:getlago/lago-api.git` at commit `71680d30cf695c86c59510dbb201a12307fe31be` (`.gitmodules:1-3`), checked out in detached-HEAD state. The chosen persistence model is a first-class Lerian fork with reviewable git history.

**Implementation vision:** Create `LerianStudio/lago-api` as a fork of `getlago/lago-api` (via `gh repo fork getlago/lago-api --org LerianStudio --clone=false`, or confirm it already exists). In the `api/` working tree, create a branch off the pinned upstream commit — name it descriptively, e.g. `lerian/premium-unlock` — add the Lerian fork as a remote, commit the Epic 1.1 (and any Task 1.2.3) changes as a single conventional commit following `api/AGENTS.md` (`feat(license): unlock premium capabilities` with a Context/Description body explaining the two levers and the rebase implication), and push the branch to the fork. Edge case — **keep the branch rebased on the pinned upstream SHA**, not on upstream `main`, so the parent submodule pointer stays a clean fast-forward from `71680d30` and future upstream bumps are an explicit, reviewed rebase. Do not force-push over any existing Lerian branch without checking its contents first.

**Files:**
- Create: branch `lerian/premium-unlock` on `LerianStudio/lago-api` containing the patched `license.rb` + `organization.rb` (+ spec edits)

**Verification:** `git -C api log --oneline -1` shows the unlock commit; `gh repo view LerianStudio/lago-api` resolves; the branch is visible on the fork.

**Done when:** the unlock commit is pushed to `LerianStudio/lago-api` on `lerian/premium-unlock`, based on the pinned upstream commit.

#### Task 1.3.2: Repoint the parent `api` submodule and commit

- [x] Done

> **Done (2026-08-17):** `.gitmodules` `api` url → `https://github.com/LerianStudio/lago-api.git` + `branch = lerian/premium-unlock`; `git submodule sync`; parent commit `089da8d` on branch `feat/unlock-lago-premium` records submodule SHA `8572b6bfd` (matches the fork). `front` left on upstream (no FE edits). Chose https (matches `gh` auth) over the original SSH form. Parent branch **not yet pushed / no PR opened** — awaiting user.

**Context:** `.gitmodules:1-3` binds path `api` to the upstream URL. The parent repo (`LerianStudio/lago`, branch `main`) records the submodule commit SHA in its tree; updating the unlock requires both the `.gitmodules` URL change and a new recorded SHA.

**Implementation vision:** Update `.gitmodules` so the `api` submodule `url` is `git@github.com:LerianStudio/lago-api.git` and add a `branch = lerian/premium-unlock` line; run `git submodule sync api`; ensure the `api/` working tree is checked out at the fork's unlock commit; then stage `.gitmodules` and the `api` submodule pointer and commit in the **parent** repo on a feature branch (not directly on `main`, per the harness git policy) with a conventional message describing the repoint. Edge case — **do not modify the `front` submodule**: no frontend changes are needed, so it stays on upstream `getlago/lago-front` to minimize divergence. Edge case — **CI/others cloning fresh**: because the fork is private-or-public under LerianStudio, confirm the URL form (SSH vs HTTPS) matches how the parent repo expects submodules to be fetched in CI before committing.

**Files:**
- Modify: `.gitmodules:1-3` (api url + branch)
- Modify: parent repo `api` submodule pointer (recorded SHA)

**Verification:** In a clean clone (or after `rm -rf api && git submodule update --init api`), the `api` tree checks out the unlocked commit and `License.premium?` logic reflects the patch; `git -C . config -f .gitmodules submodule.api.url` shows the Lerian fork.

**Done when:** a fresh submodule init reproduces the unlocked `lago-api`, and the parent repo has a branch commit recording the repoint.

---

## Phase 2 — Ship the unlock to the default/production run mode

*(Epic-level only — tasks elaborated by the executor after Phase 1 lands, against the image-build and compose wiring as they then exist.)*

### Epic 2.1: Build and publish the patched production API image [map:#4097]

**Goal:** a Lerian-hosted image built from the forked `lago-api` exists and runs the unlock (the default `docker-compose.yml` uses prebuilt `getlago/api:v1.51.0`, which does **not** include our source change).
**Scope:** the image *contract* (name/tag/build args from the fork's production `Dockerfile`) and, if requested, a one-off local `docker build && push` to unblock.
**CI ownership:** the repeatable build/publish **pipeline (GitHub Actions) is owned by a separate team — out of scope here**. We define the image contract + rebase runbook and hand the pipeline off.
**Dependencies:** Phase 1 (fork + verified source)
**Done when:** the patched image contract is defined (and, if a one-off build was requested, published) so a container from it reports `License.premium?` true and all integrations enabled.
**Status:** Done — contract in `docs/unlock-image-contract.md` (no build; CI owns publish).

### Epic 2.2: Repoint parent compose + deploy variants to the patched image [map:#4098]

**Goal:** every run mode that uses a prebuilt image points at the patched image so the unlock is live outside dev.
**Scope:** `docker-compose.yml` (the `x-backend-image` anchor at `docker-compose.yml:10-11`, reused by api/worker/clock), and the `deploy/docker-compose.{production,light,local}.yml` variants
**Dependencies:** Epic 2.1
**Done when:** `docker compose up` in default/prod mode runs the unlocked API; the `front` image stays `getlago/front` (no frontend change); a smoke check confirms premium via GraphQL.
**Status:** Done — non-breaking `LERIAN UNLOCK` swap markers at all 4 `x-backend-image` anchors (`docker-compose.yml:11`, `deploy/docker-compose.{production,light,local}.yml`); `image:` left on `getlago/api` so pulls don't break pre-publish. Actual repoint happens when CI publishes the image + finalizes the name.

### Epic 2.3: Document the unlock and the upstream-rebase runbook [map:#4099]

**Goal:** the unlock is discoverable and maintainable — future contributors know the two levers exist and how to carry them across an upstream Lago version bump.
**Scope:** parent repo docs (e.g. `docs/`), a short runbook next to this plan
**Dependencies:** Epics 2.1, 2.2
**Done when:** a document records the two touched files, the fork/branch, the image name, and the step-by-step rebase-onto-new-upstream procedure (re-apply `license.rb` + `organization.rb`, re-run Epic 1.2 verification, rebuild the image, bump the pins).
**Status:** Done — rebase runbook + compose swap list in `docs/unlock-image-contract.md`.

---

## Self-Review

| Check | Result |
|-------|--------|
| **Spec coverage** | Global-flag unlock → Task 1.1.1. Per-org integrations (reader + scopes for jobs) → Task 1.1.2. Frontend "no change" claim → verified in Task 1.2.2. Dev vs prod image-source split → Phase 1 (dev) + Phase 2 (prod images). Persistence via Lerian fork → Epic 1.3. Test-suite fallout → Task 1.2.3. Docs/rebase → Epic 2.3. No spec requirement left uncovered. |
| **Vagueness scan** | No "appropriate"/"TBD"/"handle edge cases" in detailed tasks; edge cases (test stubs, stale worker image, `super` reader, scope lambda timing, front submodule untouched) are each named with a resolution. |
| **Contract consistency** | The Lever-2 `organization.rb` block is pinned as a snippet so predicate (line 198, unchanged), reader override, and scope override agree across Tasks 1.1.2, 1.2.1, and 1.2.2. `PREMIUM_INTEGRATIONS` referenced by exact location. GraphQL operation names (`getCurrentUserInfos`, `getOrganizationInfos`) match the frontend hooks. |
| **Phase boundaries** | Phase 1 ends with a fully unlocked, verified, persisted dev stack (working software). Phase 2 ends with the unlock live in prod mode + documented. Neither ends mid-refactor. |
| **Verification plausibility** | Commands target real paths (`api/lib/lago_utils/lago_utils/license.rb`, `api/app/models/organization.rb`, `docker-compose.dev.yml`) and the container-run convention from `api/AGENTS.md` (`lago exec api ...`). |

**Note for the implementer:** these changes bypass Lago's paid licensing. Confirm the fork's license terms permit self-unlocking premium features before shipping to any environment beyond local development.
