# Penyzen — DevSecOps Tooling & Dependency Update Plan

**Scope:** the whole monorepo — developer tooling, CI/CD pipelines, security scanning, and dependency freshness (proposal only; no dependency is updated by this document)

**Purpose:** one place to see every hardening issue, its priority, the recommended fix, and how to know it is done — plus the dependency update plan we execute before production

**Priority legend:** 🔴 Critical · 🟠 High · 🟡 Medium · 🟢 Low

**How to read an issue**

- **What** — the problem.
- **Why** — impact if left alone.
- **How** — the recommended fix.
- **Where** — files and lines with the evidence.
- **Acceptance** — checks that prove the fix is complete.

**Contents**

1. [Current state](#1-current-state)
2. [Developer tooling & pre-commit](#2-developer-tooling--pre-commit)
3. [Secret scanning](#3-secret-scanning)
4. [Static analysis (SAST)](#4-static-analysis-sast)
5. [Infrastructure-as-Code scanning](#5-infrastructure-as-code-scanning)
6. [Supply chain: audit & licenses](#6-supply-chain-audit--licenses)
7. [Dependency alerts & updates](#7-dependency-alerts--updates)
8. [CI/CD pipeline hardening](#8-cicd-pipeline-hardening)
9. [Dependency update plan](#9-dependency-update-plan)
10. [What to fix first](#10-what-to-fix-first)

---

## 1. Current state

Verified against the repository at the time of writing:

- **Pre-commit:** nothing. No husky, no lint-staged, no commit hooks of any kind.
- **Formatting:** Prettier 3 is installed with a `.prettierrc`, but there is no `format` script, no CI format check, and no `eslint-config-prettier` — formatting is unenforced and can conflict with ESLint.
- **Lint coverage:** the root script is `eslint packages services --ext .ts` — `apps/web`, `infra`, and the root-level test folders are never linted by `npm run lint`. The web app has its own minimal `next/core-web-vitals` config.
- **Secret scanning:** none. No gitleaks, no GitHub secret scanning configuration, nothing in CI.
- **SAST:** none. No SAST engine (CodeQL, SonarQube, or Semgrep), no security-oriented ESLint plugins.
- **IaC scanning:** none. No cdk-nag, no cfn-lint; the CDK has no assertion tests at all (see backend TST-5).
- **Dependency auditing:** no `npm audit` gate in CI, no license check, no Dependabot/Renovate configuration.
- **CI (`ci.yml`):** install → build internal packages → generate Prisma client → typecheck → lint → build → test. No scanning steps of any kind.
- **Deploy (`deploy-backend.yml`):** OIDC deploy → CDK deploy → invoke migrator → smoke tests. No security gate between a merge and a production deploy.
- **Actions and runtime pinning:** workflows use floating major tags (`@v5`); CI runs Node 24 while `engines` only says `>=22` and no `.nvmrc` exists.

---

## 2. Developer tooling & pre-commit

#### DEV-1 — Run husky + lint-staged on every commit

**Priority:** 🟠 High

**What:** There are no git hooks. Nothing stops a commit that fails lint, is unformatted, or contains a secret.

**Why:** Broken or dirty code lands on `dev` and surfaces only when CI runs — or worse, after review has already spent cycles on it.

**How:** Add `husky` and `lint-staged`. The pre-commit hook runs lint-staged, which for staged files runs `eslint --fix` and `prettier --write` (and typecheck on staged TypeScript where cheap). Keep the hook fast — it must never run the full monorepo build. Optionally add a pre-push hook that runs the unit suite for touched workspaces.

**Where:** `package.json:25-33` (no hook tooling in devDependencies); no `.husky/` directory exists

**Acceptance:**

- [ ] `npm run prepare` installs husky on `npm ci` (fresh clone included).
- [ ] A staged file that fails ESLint cannot be committed without an explicit bypass.
- [ ] lint-staged formats only staged files; a full commit cycle stays under a few seconds.
- [ ] The hooks work on Windows (the project is developed on win32 — no bash-only assumptions).

#### DEV-2 — Enforce Prettier and close the lint-coverage gaps

**Priority:** 🟠 High

**What:** Prettier is configured but never enforced; `eslint-config-prettier` is missing so ESLint and Prettier can disagree. The root lint script covers only `packages` and `services` `.ts` files — `apps/web`, `infra`, and root tests are outside it.

**Why:** Formatting drifts per author and per machine. Lint gaps mean whole workspaces ship with no static analysis at all, including the CDK infrastructure code.

**How:** Add `format` and `format:check` scripts at the root; add `eslint-config-prettier` to the ESLint config so the two tools never fight; run `format:check` in CI. When the ESLint 9 flat config lands (DEP-2), unify coverage so `npm run lint` reaches `apps/web`, `infra`, and test folders (keep `next lint` rules for the web app via its shareable config). Keep the typed-lint heap workaround documented until the config is sharded.

**Where:** `package.json:14` (lint script); `.eslintrc.js:1-22`; `.prettierrc:1-7`; `apps/web/.eslintrc.json:1-5`; `.github/workflows/ci.yml:44-48`

**Acceptance:**

- [ ] `npm run format:check` passes on a clean tree and fails on unformatted files.
- [ ] CI fails when a file is not Prettier-formatted.
- [ ] `eslint-config-prettier` is the last extend in every ESLint config.
- [ ] `npm run lint` covers `packages`, `services`, `apps/web`, `infra`, and root test folders.

#### DEV-9 — Enforce commit conventions (optional)

**Priority:** 🟢 Low

**What:** Commit messages are free-form; there is no convention or commit-msg hook.

**Why:** Without a convention, changelogs and release notes cannot be generated, and history is hard to scan. This is optional — adopt it only if the team wants automated releases.

**How:** If adopted, use Conventional Commits with `commitlint` in a husky `commit-msg` hook. Keep the rule set minimal (`type-enum`, subject-case off).

**Where:** `package.json` (no commitlint dependencies)

**Acceptance:**

- [ ] Either a commit-msg hook enforces the convention, or the team explicitly records the decision not to.

#### DEV-10 — Pin the Node version everywhere

**Priority:** 🟢 Low

**What:** `engines` says `node >=22`, CI runs Node 24, and there is no `.nvmrc` / `.node-version`. Local machines can drift from CI silently. Node 24 is the agreed floor — it is what CI runs and what the LTS cadence supports through launch.

**Why:** "Works on my machine" failures and subtle ABI differences (especially with native modules like `sharp` and Prisma engines).

**How:** Set `engines` to `node >=24` (and `npm >=10`) so an outdated local runtime refuses to install instead of failing mysteriously later. Add `.nvmrc` and `.node-version` with `24` so version managers pick it up automatically, and add `devEngines` guidance for npm 10+. Update the CI `node-version` input to read the pin file so the two can never diverge.

**Where:** `package.json:34-37`; `.github/workflows/ci.yml:21-24`

**Acceptance:**

- [ ] `engines` requires `>=24`; installing on Node 22 fails with a clear engine error.
- [ ] `.nvmrc` and `.node-version` exist, contain `24`, and match CI.
- [ ] Fresh clone on a version-manager picks up the pinned version automatically.

---

## 3. Secret scanning

#### DEV-3 — Scan for secrets with gitleaks

**Priority:** 🟠 High

**What:** No tool scans the working tree, commits, or history for secrets. A committed `.env`, AWS key, or Stripe live key would land unnoticed.

**Why:** The stack is dense with secret material — Stripe keys, Cognito config, RDS credentials, SSM parameters, OIDC role ARNs. One leaked live key is a production incident.

**How:** Adopt gitleaks as the repo-local baseline (it works with no GitHub add-ons):

1. **History scan once** on adoption: `gitleaks detect --redact --log-opts=--all` — triage anything it finds before it becomes false lore.
2. **Pre-commit hook** via the husky setup from DEV-1: `gitleaks protect --staged --redact`.
3. **CI job** on every PR and a scheduled (weekly) full-history run.
4. **`.gitleaks.toml`** with a narrow allowlist for known placeholders only (e.g., the `pk_test_` / `replace-me` values in `.env.example` and CDK context) — never allowlist a path wholesale.

Additionally enable GitHub's secret scanning and push protection in repo settings (free for public repositories; requires GitHub Advanced Security for private ones) as defense in depth.

**Where:** no gitleaks config exists; `.env.example`; `infra/lib/stacks/api-stack.ts` (placeholders per SEC-3)

**Acceptance:**

- [ ] Full-history scan has been run and every finding is either fixed or documented as a false positive in `.gitleaks.toml`.
- [ ] A staged secret cannot be committed (hook test with a fake key).
- [ ] CI fails on a PR that introduces a plausible secret.
- [ ] The scheduled scan opens an issue or fails loudly on a finding.

---

## 4. Static analysis (SAST)

#### DEV-4 — Add SAST: CodeQL (or SonarQube) + ESLint security plugins

**Priority:** 🟠 High

**What:** There is no SAST beyond the base TypeScript ESLint rules. Injection, unsafe `eval`, prototype pollution, and regex DoS patterns are not flagged.

**Why:** These are exactly the classes of bug that reach production in API layers — string-built SQL fragments, `dangerouslySetInnerHTML`, unsanitized redirects.

**How:**

- **ESLint layer (always on):** add `eslint-plugin-security` and `eslint-plugin-no-unsanitized` (plus `eslint-plugin-jsx-a11y` is already partially covered in web via `next/core-web-vitals`). These run inside the existing lint step — zero new infrastructure.
- **Engine layer — pick one:**
  - **CodeQL (preferred where available):** add a `.github/workflows/codeql.yml` running the `github/codeql-action` for `javascript-typescript` on PRs into `dev`/`master` and weekly. CodeQL for private repositories requires GitHub Advanced Security.
  - **SonarQube / SonarCloud (if GitHub does not provide SAST):** run analysis either through the SonarScanner in CI or Sonar's GitHub Action, with a quality gate on new code (security hotspots, vulnerabilities, maintainability, test coverage). SonarCloud is free for public repositories; SonarQube Community Edition is free to self-host. It decorates PRs with per-issue findings, which CodeQL-only setups get for free via GitHub code scanning.
  - **Semgrep OSS (lightest fallback):** if neither CodeQL nor a Sonar instance is available, Semgrep's TypeScript ruleset with a small custom rules file covers the same must-catch classes.

  Whichever engine is chosen, record the decision (and why the others were not chosen) in the PR that adds this — the requirement is *one enforced engine plus the ESLint layer*, not a specific vendor.
- **Where it gates:** high/critical findings fail the PR check, same as lint.

**Where:** `.eslintrc.js:6-13` (extends list, no security plugins); `.github/workflows/` (only `ci.yml`, `deploy-backend.yml`, `enforce-source-branch.yml`)

**Acceptance:**

- [ ] `eslint-plugin-security` and `eslint-plugin-no-unsanitized` run in CI and existing code is clean or the findings are triaged.
- [ ] One SAST engine — CodeQL, SonarQube/SonarCloud, or Semgrep — runs on PRs and weekly; high-severity findings block merge.
- [ ] The chosen engine's quality gate / alert rules are configured in code (workflow file or `sonar-project.properties`), not only in a UI.
- [ ] A deliberately vulnerable snippet (test fixture) is caught by the pipeline.

---

## 5. Infrastructure-as-Code scanning

#### DEV-5 — Scan infrastructure with cdk-nag and cfn-lint

**Priority:** 🟠 High

**What:** The CDK stacks have no assertion tests and no best-practice scanning. Wildcard IAM resources (SEC-4), disabled CloudTrail/encryption checks, and public-access settings can ship silently — the review already found several such gaps by hand.

**Why:** Infrastructure bugs are found in production or in an incident, never in review, unless a machine looks for them.

**How:**

- **cdk-nag:** add `cdk-nag` with the `AwsSolutionsChecks` pack (and `NIST80053R5Checks` if targeting compliance) as assertions over the synthesized template in the infra test suite (this lands together with backend TST-5). Every suppression must carry an explicit `// reason:` annotation; CI fails on unsuppressed High-severity findings.
- **cfn-lint:** run `cfn-lint` against `cdk synth` output in CI as a fast second opinion.
- Keep the two cheap enough to run on every PR that touches `infra/**`.

**Where:** `infra/` (no test files exist); `infra/lib/stacks/api-stack.ts`; `infra/lib/stacks/database-stack.ts`; `infra/lib/stacks/storage-stack.ts`

**Acceptance:**

- [ ] `cdk-nag` assertions run in CI; current findings (wildcard IAM, etc.) are either fixed or annotated with reasons.
- [ ] `cfn-lint` runs on synthesized output in CI.
- [ ] A new stack without nag assertions fails the infra test job.

---

## 6. Supply chain: audit & licenses

#### DEV-6 — Gate dependencies: npm audit + license check

**Priority:** 🟠 High

**What:** CI never runs `npm audit`, and no license policy exists. Known-vulnerable or copyleft-licensed packages can enter through any `npm install`.

**Why:** The dependency tree includes AWS SDKs, Stripe, Prisma, and the whole Next/React surface — a critical advisory in any of them is a production risk with no current tripwire.

**How:**

- Add a CI step `npm audit --audit-level=high` (fail the job on high/critical). Maintain a small `audit-exceptions.json` (or use `overrides`) for findings that are not exploitable in our context; every exception carries a comment and an expiry date, reviewed monthly.
- Add a license check (e.g., `license-checker-rseidelsohn --onlyAllow 'MIT;Apache-2.0;BSD-2-Clause;BSD-3-Clause;ISC;0BSD;Unlicense;CC0-1.0'`) as a CI step. This catches accidental GPL/unknown-license additions before review.
- Wire both as separate parallel jobs so a slow audit never blocks lint feedback.

**Where:** `.github/workflows/ci.yml:15-63` (no audit or license steps); `package-lock.json`

**Acceptance:**

- [ ] CI fails on a newly introduced high/critical advisory.
- [ ] Every audit exception is documented with a reason and review date.
- [ ] CI fails on a disallowed license.

---

## 7. Dependency alerts & updates

#### DEV-7 — Stay current with Dependabot

**Priority:** 🟠 High

**What:** There is no Dependabot or Renovate configuration. Security advisories and new releases surface only when someone happens to read a changelog.

**Why:** The lockfile already contains a deprecated, vulnerable `next@14.2.13` and an EOL `eslint@8` (see section 9) that no alert flagged. This is how the tree drifts.

**How:** Add `.github/dependabot.yml`:

- `package-ecosystem: npm` — weekly schedule, directory `/`, with **grouped** minor/patch updates per scope (aws-sdk, eslint toolchain, react/next) and major updates left ungrouped for deliberate review.
- `package-ecosystem: github-actions` — weekly, so action pins (DEV-8) stay fresh.
- Enable **Dependabot security alerts** and **Dependabot auto-rebase** in repo settings.
- Merge rules: green CI + smoke tests; majors follow the process in section 9.

Dependabot is the baseline because it is native and zero-infrastructure. If grouped-update needs outgrow it (e.g., custom scheduling, osvnr filtering), migrate to Renovate without changing the process rules.

**Where:** no `.github/dependabot.yml` exists

**Acceptance:**

- [ ] `dependabot.yml` exists for npm and github-actions and opens PRs on the weekly schedule.
- [ ] Security alerts are enabled; a published advisory produces a Dependabot PR within days.
- [ ] The team has a standing rule for who reviews/merges Dependabot PRs.

---

## 8. CI/CD pipeline hardening

#### DEV-8 — Harden the CI/CD pipelines

**Priority:** 🟠 High

**What:** Workflows pin actions only by floating major tag (`actions/checkout@v5`, `aws-actions/configure-aws-credentials@v5`), CI has no concurrency cancellation, the scan steps from DEV-3..6 have no home yet, and nothing records that the CI check is *required* in branch protection.

**Why:** A compromised or silently re-tagged action runs with the workflow's privileges — including the OIDC deploy role in `deploy-backend.yml`. Without required checks, any merge path bypasses CI entirely.

**How:**

- **Pin actions to full commit SHAs** with a trailing version comment (e.g., `uses: actions/checkout@<sha> # v5.x`). Dependabot (DEV-7) keeps the pins updated.
- **Least privilege:** keep `permissions: contents: read` at workflow level; grant `id-token: write` only on the deploy job (already the case — preserve it).
- **Concurrency:** add a `concurrency` group with `cancel-in-progress: true` to `ci.yml` for PR runs so superseded commits stop burning minutes.
- **Gate the deploy:** insert the gitleaks (DEV-3) and `npm audit` (DEV-6) steps into `deploy-backend.yml` before `cdk deploy` so nothing unscanned reaches an account.
- **Required checks:** wire `CI / check` as a required status check for `dev` and `master` in branch protection — the ci.yml header already documents this intent; it must be actually configured.
- **Split jobs:** run lint / typecheck / test / scan as parallel jobs with a shared install+build step so total wall-clock time goes down as scanners are added.

**Where:** `.github/workflows/ci.yml:19-24,41-63`; `.github/workflows/deploy-backend.yml:58-72,105-146`; `.github/workflows/enforce-source-branch.yml`

**Acceptance:**

- [ ] Every `uses:` is pinned to a commit SHA with a version comment.
- [ ] Superseded PR runs are cancelled automatically.
- [ ] The deploy workflow fails before `cdk deploy` if gitleaks or audit fails.
- [ ] Branch protection requires the CI check on `dev` and `master` (verified in repo settings).

---

## 9. Dependency update plan

This is a **proposal** — no dependency is updated by this document. Versions below were read from `package-lock.json`; exact latest versions are verified with `npm outdated` (registry access required) at execution time. Cross-references to backend/frontend review issues are deliberate: several updates unblock or depend on those fixes.

#### DEP-1 — Patch the known-vulnerable and deprecated packages first

**Priority:** 🔴 Critical — do first

**What:** The lockfile itself flags two problems: `next@14.2.13` is marked deprecated with a security vulnerability (December 2025 advisory), and `eslint@8.57.1` is end-of-life. `@types/handlebars@4.1.0` is a deprecated stub (Handlebars ships its own types).

**Why:** A known-vulnerable Next.js version is the single most exposed dependency — it serves the entire public web app. An EOL linter receives no security fixes and blocks every other tooling upgrade.

**How:**

- Upgrade Next.js to the **patched 14.2.x line first** (smallest possible diff that removes the advisory), then plan the major move in DEP-4.
- Remove `@types/handlebars` immediately (zero-risk; Handlebars ≥4.7 bundles its types).
- ESLint 8 → 9 happens as one unit with DEP-2 below, because the EOL version and the flat-config migration cannot be sequenced apart cleanly.

**Where:** `apps/web/package.json:23,40` (`next`, `eslint-config-next`); root `package.json:26-27`; `packages/email-templates/package.json:18`

**Acceptance:**

- [ ] `npm audit --audit-level=high` is clean for the web app's production tree.
- [ ] No package in the lockfile carries a deprecation notice.
- [ ] Full CI + smoke tests green after the patch upgrade.

#### DEP-2 — Modernize the lint toolchain (ESLint 9 + typescript-eslint 8)

**Priority:** 🟠 High

**What:** ESLint 8.57 (EOL) with `@typescript-eslint@7` and the legacy `.eslintrc.js` format. The current major line is ESLint 9 with flat config and `@typescript-eslint@8`.

**Why:** The linter is the backbone of DEV-1/DEV-2/DEV-4; staying on EOL tooling means the whole hardening stack sits on an unsupported foundation, and typed-lint performance fixes land only on the new line.

**How:** One migration PR: convert `.eslintrc.js` and `apps/web/.eslintrc.json` to `eslint.config.mjs` (flat), upgrade `eslint` + `@typescript-eslint/*` + `eslint-config-next` to compatible majors, add `eslint-config-prettier` and the security plugins (DEV-4) in the same pass, and fix any new findings. Keep the web app's `next/core-web-vitals` rules via the flat-config export from `eslint-config-next`.

**Where:** `.eslintrc.js:1-22`; `apps/web/.eslintrc.json:1-5`; root `package.json:26-27`

**Acceptance:**

- [ ] Only flat config files remain; no `.eslintrc*` files.
- [ ] Lint coverage matches DEV-2 acceptance (web, infra, tests included).
- [ ] CI lint step passes with the new config; new findings are fixed or explicitly waived.

#### DEP-3 — Upgrade Prisma to 7.10.x

**Priority:** 🟠 High

**What:** The project runs Prisma 6.19 (`prisma` and `@prisma/client`). The agreed target is the **7.10.x** line.

**Why:** Prisma 7 is the supported major with performance and query-engine improvements; staying on 6.x means the database foundation ages before launch. This upgrade must be sequenced with the database work already planned (DAT-0 generate-in-build, DAT-1 Prisma Migrate) so the schema toolchain is rebuilt once, not twice.

**How:** Plan as a dedicated migration: read the 6→7 breaking-changes guide at execution time (generator config, `prisma-client-js` output location, driver-adapter and engine changes), regenerate the client with both binary targets preserved, re-baseline the Lambda layer build (DAT-7 / `deploy-backend.yml` layer step), and run the full migration suite against a disposable database (DAT-6 harness). Do not combine with feature work.

**Where:** `packages/database/package.json:17-23`; `packages/database/prisma/schema.prisma:9-13`; `.github/workflows/deploy-backend.yml:81-91`

**Acceptance:**

- [ ] `prisma` and `@prisma/client` are on 7.10.x; the generated client builds in CI with no manual step (DAT-0).
- [ ] Migrations apply cleanly on a disposable Postgres; smoke tests pass post-deploy.
- [ ] The Prisma Lambda layer build step works with the 7.x client layout.

#### DEP-4 — Upgrade React 19 + Next.js (major)

**Priority:** 🟠 High

**What:** React 18.3.1 and Next 14.2.x; the current majors are React 19 and Next 15+ (verify the exact target at execution time).

**Why:** React 19 brings the compiler-era performance and hook improvements the "responsive, low-connection" goal depends on; Next 14 is already carrying a security advisory (DEP-1) and its LTS window is finite.

**How:** Sequence after DEP-1 (patched baseline) and together, since Next and React majors couple. Check, in order: Amplify adapter compatibility (`@aws-amplify/adapter-nextjs`), `eslint-config-next`, `react-easy-crop`, `@stripe/react-stripe-js`, and `@aws-amplify/ui-react` for React 19 support. Upgrade the web app in one PR with the existing CI placeholder build (ci.yml) and Amplify preview as the verification environments. Keep the frontend data-layer migration (review P2/P7) in the same window so TanStack Query's React-19-compatible line lands once.

**Where:** `apps/web/package.json:13-31`; `.github/workflows/ci.yml:52-60`

**Acceptance:**

- [ ] `next build` and Amplify deploy both succeed on the new majors.
- [ ] No React 19 peer-dep warnings remain in the lockfile.
- [ ] Donate, auth, and admin flows pass smoke tests on a deployed preview.

#### DEP-5 — Upgrade Zod to 4

**Priority:** 🟠 High

**What:** Zod 3.25 across `packages/shared`, every backend service, and the web app; Zod 4 is the current major with a faster parser and stricter defaults.

**Why:** Zod is the contract layer (request schemas, env validation, and — per frontend P1 — the shared frontend schemas). Upgrading early avoids doubling the migration surface once more code adopts schemas.

**How:** Upgrade `zod` in `packages/shared` first and rebuild the dependents; fix deprecations (e.g., `.trim()` refinements, error-message customizations, `z.infer` edge cases). Because services and web import schemas from shared, one coordinated version bump across the workspace keeps a single Zod instance in the tree.

**Where:** `packages/shared/package.json:14`; `services/*/package.json` (zod entries); `apps/web/package.json:29`

**Acceptance:**

- [ ] One Zod major (`^4`) across the workspace; no duplicate major versions in the lockfile.
- [ ] Schema-driven validation tests and smoke tests pass unchanged.

#### DEP-6 — Remaining majors and tooling alignment

**Priority:** 🟡 Medium

**What:** A set of packages that are behind their current majors or superseded, none individually urgent but collectively aging: `stripe@16`, `tailwindcss@3` (4 is current), `esbuild@0.23`, `@types/node@20` (the runtime floor is Node 24 per DEV-10), `ts-node@10` (superseded by `tsx`), `aws-jwt-verify@4`, and the `@aws-sdk/*` v3 minor line.

**Why:** Accumulated majors turn into one impossible mega-upgrade later; `ts-node` in particular is the only reason `db:seed` (DAT-4) and Prisma script tooling carry a legacy dependency.

**How:** Handle as small grouped PRs, each independently green: Stripe SDK major (verify webhook-signature API unchanged — pairs with backend SEC-8); Tailwind 4 (CSS-first config migration — pairs with frontend P6 shadcn adoption); esbuild across services (build tooling only); `@types/node` → 24 to match the Node 24 floor; replace `ts-node` with `tsx` for scripts and remove it; bump `aws-jwt-verify` and the AWS SDK line together with a dependency-review pass. Keep `vitest@4`, `prettier@3`, `typescript@5.9`, `pg@8`, `sharp@0.35`, `handlebars@4.7` as-is — they are current.

**Where:** `services/payment-service/package.json:15`; `services/user-service/package.json:20`; `apps/web/package.json:22,29,35-43`; `services/*/package.json` (esbuild devDeps); `packages/database/package.json:23` (`ts-node`); `infra/package.json:14-25` (`aws-cdk` vs `aws-cdk-lib` alignment)

**Acceptance:**

- [ ] Each group lands as its own PR with green CI and smoke tests.
- [ ] `ts-node` is gone; scripts run with `tsx`.
- [ ] `@types/node` matches the Node 24 floor; no stale engine types remain.

#### DEP-7 — Remove dead and stub dependencies

**Priority:** 🟢 Low

**What:** Beyond the deprecated stub in DEP-1, the review found unused dependencies: `@penyzen/shared` in email-templates, and `zod` + `@aws-amplify/ui-react` declared in web but unused. The deprecated transitive tree (`glob@7`, `inflight`, `rimraf@3`, `@humanwhocodes/*`) disappears as a side effect of DEP-2.

**Why:** Dead weight obscures real audit results and slows installs. `@aws-amplify/ui-react` deserves a specific note: it has **zero imports** across `apps/web/src` (verified) — the auth pages are custom-built with `aws-amplify/auth` + Tailwind, so its prebuilt components (`Authenticator` etc.) buy nothing here. It ships a large component and theming layer that would fight Tailwind once shadcn/ui lands (frontend P6), and it drags the first-load bundle down for no user-visible benefit. It is purely presentational: removing it does **not** touch Amplify auth itself — `aws-amplify` core (sign-in, tokens, Cognito, server auth) and `@aws-amplify/adapter-nextjs` (`src/lib/amplify-server.ts`) are in active use and stay.

**How:** Execute the existing backend issue STR-8 as part of this plan rather than separately; verify usage with `depcheck` or knip before each removal. Remove `@aws-amplify/ui-react` from `apps/web/package.json` — no code changes needed because nothing imports it. Keep `zod` in web: the frontend form migration (P3) consumes it. The UI direction remains shadcn/ui (frontend review P6 and architecture doc section 9).

**Where:** `packages/email-templates/package.json:13-16`; `apps/web/package.json:14` (`@aws-amplify/ui-react`); `apps/web/package.json:29` (`zod` — keep); cross-ref backend STR-8, frontend P3/P6

**Acceptance:**

- [ ] No unused direct dependencies remain (checked with knip/depcheck).
- [ ] `@aws-amplify/ui-react` is gone from `package.json` and the lockfile; auth pages, sign-in, and sign-out still work (they never imported it).
- [ ] `aws-amplify` core and `@aws-amplify/adapter-nextjs` remain — Amplify auth is unaffected.
- [ ] The lockfile no longer contains `inflight`, `glob@7`, `rimraf@3`, or `@humanwhocodes/*`.

### Process rules for every dependency PR

1. **One major per PR.** Minor/patch updates may be grouped by scope (Dependabot groups).
2. **Green CI and smoke tests before merge.** The post-deploy smoke suite (deploy-backend.yml) is the final gate for majors that touch runtime behavior.
3. **Rollback is a revert.** No in-place hotfixes on a failing upgrade PR; revert and re-attempt with a smaller diff.
4. **Lockfile commits only via npm.** Never hand-edit `package-lock.json`.
5. **Security patches skip the queue.** A high/critical advisory in a direct dependency is patched immediately on its current major, even outside this plan.

---

## 10. What to fix first

Tooling first — it is the machinery every later fix relies on.

| Order | Focus | Issues |
|---|---|---|
| 1 | Hooks & format baseline | DEV-1, DEV-2 |
| 2 | Secrets & dependency alerts | DEV-3, DEV-7 |
| 3 | Vulnerable packages | DEP-1 (Next patch, remove stub) |
| 4 | Scanners in CI | DEV-4, DEV-5, DEV-6, DEV-8 |
| 5 | Lint toolchain major | DEP-2 |
| 6 | Foundation majors | DEP-3 (Prisma 7.10), DEP-4 (React/Next), DEP-5 (Zod 4) |
| 7 | The rest | DEP-6, DEP-7, DEV-9, DEV-10 |

Note for sequencing with the backend review: DEP-3 (Prisma) pairs with DAT-0/DAT-1/DAT-7; DEP-4 pairs with frontend P2/P7 (TanStack Query) so the React-19-compatible data layer lands once; DEP-2 is the vehicle for the DEV-4 ESLint plugins and the DEV-2 lint-coverage expansion.
