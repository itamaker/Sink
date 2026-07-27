# Cloudflare Workers Migration Design

## Context

The production Sink instance is currently deployed through Cloudflare Pages from
the `itamaker/Sink` repository. Pushes to `master` trigger Pages builds, and the
custom domain `link.jiazhaoyang.com` points to the last successful Pages
deployment.

The live Pages configuration predates the D1-backed storage architecture. It has
the existing `sink` KV namespace, Analytics Engine dataset, R2 bucket, and
Workers AI binding, but no `DB` binding or Sink D1 database. The current
repository already contains the Worker entrypoint, bindings, migration tooling,
and `pnpm deploy:worker` command needed for a Workers deployment.

## Goals

- Replace Cloudflare Pages with a Cloudflare Worker named `sink`.
- Preserve automatic production deployments when `master` is pushed.
- Preserve the current site token, analytics configuration, KV data, R2 data,
  and custom domain.
- Create D1 as the authoritative link store and migrate legacy KV links safely.
- Avoid user-visible downtime until the final custom-domain cutover.
- Keep a recoverable rollback path throughout the migration.

## Non-goals

- No application feature or UI changes.
- No unrelated Cloudflare resource cleanup.
- No non-production branch deployment workflow in the initial migration.
- No deletion of the Pages project or legacy KV data.

## Selected Architecture

Use Cloudflare Workers Builds with the existing GitHub repository and Cloudflare
GitHub installation.

Workers Builds will listen to the `master` branch and run:

```text
Build command:  pnpm build
Deploy command: pnpm deploy:worker
```

`pnpm build` produces the Nuxt/Nitro Worker entrypoint and static assets. The
Pages-only `postbuild` migration guard does not run because Workers Builds does
not set `CF_PAGES=1`. The deploy command then:

1. Generates the gitignored `wrangler.deploy.jsonc` from Cloudflare build
   variables.
2. Applies remote D1 migrations.
3. Deploys `.output/server/index.mjs` and `.output/public` through Wrangler.

The Worker name must remain `sink`, matching `wrangler.jsonc`.

## Cloudflare Resources

Create one new D1 database named `sink`. Reuse the existing resources already
attached to the Pages project:

| Binding | Resource | Migration action |
| --- | --- | --- |
| `DB` | New D1 database named `sink` | Create and bind |
| `KV` | Existing KV namespace named `sink` | Reuse unchanged |
| `ANALYTICS` | Existing dataset named `sink` | Reuse unchanged |
| `R2` | Existing bucket named `sink` | Reuse unchanged |
| `AI` | Workers AI | Reuse |
| `ASSETS` | Worker static assets | Created by Wrangler configuration |

Resource identifiers must be resolved from the Cloudflare account during
execution and stored in Workers Builds settings. They must not be committed to
the repository.

## Build Configuration

Configure these Workers Builds variables:

| Name | Type | Source |
| --- | --- | --- |
| `DEPLOY_D1_DATABASE_ID` | Plain build variable | UUID of the new `sink` D1 database |
| `DEPLOY_KV_NAMESPACE_ID` | Plain build variable | ID of the existing `sink` KV namespace |
| `DEPLOY_R2_BUCKET_NAME` | Plain build variable | `sink` |
| `DEPLOY_ANALYTICS_DATASET` | Plain build variable | `sink` |

`DEPLOY_D1_DATABASE_NAME` is omitted because its repository default is `sink`.
`DEPLOY_KV_PREVIEW_NAMESPACE_ID` and
`DEPLOY_R2_PREVIEW_BUCKET_NAME` are omitted because non-production branch builds
are not enabled in the initial migration.

Workers Builds creates and manages its own deployment token. No
`CLOUDFLARE_API_TOKEN` build variable is required.

## Runtime Configuration

Configure the Worker with:

| Name | Type | Migration action |
| --- | --- | --- |
| `NUXT_SITE_TOKEN` | Encrypted secret | Copy the existing Pages value exactly |
| `NUXT_CF_ACCOUNT_ID` | Plain variable | Preserve the existing account ID |
| `NUXT_CF_API_TOKEN` | Encrypted secret | Copy the existing analytics token exactly |

Sensitive values must be transferred without printing them to logs, command
output, commits, or design artifacts.

## Migration Sequence

1. Create the `sink` D1 database.
2. Create the `sink` Worker control-plane record and configure its runtime
   variables and secrets. Resource bindings are materialized by the first
   deployment from the generated Wrangler configuration.
3. Connect `itamaker/Sink` to Workers Builds with `master` as the production
   branch and configure the build variables and commands above.
4. Trigger the initial Worker build without changing the Pages project or
   custom domain.
5. Verify the Worker on its `workers.dev` hostname:
   - the root page and dashboard load;
   - authentication accepts the existing site token;
   - bindings are present;
   - D1 migrations completed;
   - logs show no startup or request errors.
6. Temporarily avoid dashboard/API link edits while storage migration and
   cutover are in progress. Public redirects may continue through Pages.
7. Open Dashboard → Links on the Worker deployment to start or resume the
   legacy KV-to-D1 migration. Monitor Dashboard → Migrate → D1 until complete.
8. Verify representative existing links and perform one reversible
   create/open/edit/delete test against the Worker.
9. Detach `link.jiazhaoyang.com` from Pages and attach it as a Worker custom
   domain.
10. Verify TLS, the dashboard, representative redirects, analytics, and Worker
    logs through the custom domain.
11. Disable Pages production auto-deployments. Keep the Pages project and its
    `pages.dev` hostname temporarily for audit and pre-cutover rollback.

## Failure Handling and Rollback

- If D1 creation, Worker deployment, Git connection, or the initial build
  fails, make no Pages or domain changes.
- If KV-to-D1 migration fails, keep Pages serving production, retain the
  original KV data, correct the failing record or configuration, and retry the
  migration.
- If custom-domain attachment fails before new D1-only writes occur, reattach
  the domain to the last successful Pages deployment.
- After the Worker accepts production writes, rollback should use Cloudflare
  Worker version rollback rather than the old Pages deployment, because the old
  Pages version is not authoritative for new D1 records.
- Do not delete the KV namespace, D1 database, Pages project, or R2 bucket as
  part of this migration.

## Verification

The migration is complete when:

- a push to `master` produces a successful Workers Build and deployment;
- `link.jiazhaoyang.com` resolves to the `sink` Worker;
- the existing site token still authenticates;
- the D1 migration reports completion;
- representative legacy links redirect correctly;
- a temporary link passes create, read, edit, redirect, and delete checks;
- Analytics, R2, AI, and scheduled backup bindings are present;
- Worker logs contain no startup or request errors during the verification
  window;
- Pages production auto-deployments are disabled only after the Worker passes
  all checks.
