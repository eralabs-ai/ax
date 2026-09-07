# Releasing `ax`

Releases are automatic and run entirely in CI. Nobody publishes from a laptop,
and nobody edits `package.json`'s version by hand.

## How a release happens

1. Land changes on `main` through normal PRs with conventional titles
   (`feat:`, `fix:`, `chore:`, …). The `Conventional commit title` check
   enforces this; the title becomes the squash commit on `main`.
2. On every push to `main`, [release-please](https://github.com/googleapis/release-please)
   opens or updates **one** PR titled `chore(main): release X.Y.Z`. It bumps
   `package.json`, writes the `CHANGELOG.md` entry from the commit titles, and
   picks the version from the commit types:
   - `fix:` → patch
   - `feat:` → minor (we are pre-1.0, so `bump-minor-pre-major` keeps
     breaking changes at minor too)
   - `chore:`, `docs:`, `ci:`, `test:`, `refactor:` → no release on their own
3. **Merging the release PR is the release.** The push that merges it makes
   release-please tag the commit `vX.Y.Z` and create the GitHub Release, then
   the `publish` job in [release.yml](.github/workflows/release.yml) lints,
   typechecks, tests, builds, smoke-tests the bundled binary, verifies the
   tarball, and runs `npm publish --provenance` via OIDC Trusted Publishing.
4. If the release changes the command/flag surface the agent-ready-website
   skill invokes (today: `audit` and the flags its playbook shows), bump the
   pinned `CLI_RANGE` in the main repo's
   `src/lib/mcp/skills-content/agent-ready-website.ts` in the same breath and
   run `npm run skills:gen` there - the skill's `npx ax@<range>` calls
   resolve only the release line the playbook documents, so a range left
   behind quietly routes agents to the API fallback instead of the new CLI.

Nothing is left over after a release. The tag, the GitHub Release, the
changelog and the `package.json` bump all live on `main` before npm is touched,
and `publish` never commits anything, so there is no loop and no trailing PR.

To ship a version release-please would not pick on its own (e.g. `1.0.0`), put
`Release-As: 1.0.0` in the body of any commit that lands on `main`, or add a
`release-as` field to `release-please-config.json` for one release and remove it
after.

Rehearse the publish half with **Actions → Release → Run workflow** and
`dry_run` checked. It runs the full gate against current `main` and ends in
`npm publish --dry-run`; nothing is published, tagged or committed.

## One-time setup

### A token for the release PR

PRs opened with the built-in `GITHUB_TOKEN` never trigger `pull_request`
workflows (GitHub's guard against recursive runs). Without a real token the
release PR arrives with its required checks stuck at "expected", and someone
has to close and reopen it before it can merge.

Create a **fine-grained personal access token** (a maintainer's, or a machine
user's) scoped to this repository with `Contents: Read and write` and
`Pull requests: Read and write`, and store it as the repo secret
`RELEASE_PLEASE_TOKEN`. The workflow falls back to `GITHUB_TOKEN` when the
secret is absent, so nothing breaks, but the close/reopen dance returns.

If the release PR should merge on its own once green, enable auto-merge on it
once (**Settings → General → Allow auto-merge** is already on for this repo).

### OIDC Trusted Publishing

Publishing is tokenless: npm verifies the workflow's identity through GitHub's
OIDC provider. The trusted publisher is configured on npmjs.com → `ax` →
**Settings → Trusted Publisher**: GitHub Actions, org `ora`, repository
`ax`, workflow `release.yml`, no environment, `npm publish` allowed. The
`publish` job's `id-token: write` permission plus npm ≥ 11.5.1 (the job
upgrades npm explicitly — Node 22 bundles npm 10, which silently skips the
OIDC exchange) are the only other requirements.

The workflow **filename** is part of that binding. Renaming `release.yml`
rejects every publish until npmjs.com is updated to match. Same if you ever add
an `environment:` to the `publish` job (see hardening below).

If tokenless publishing is ever broken and a release can't wait, the fallback
is a **granular access token** with read+write on the `ax` **package** — `ax`
is unscoped, so a token limited to the `@ora-ai` scope does *not* cover it —
stored as `NPM_TOKEN` and passed as `NODE_AUTH_TOKEN` in the publish step.
Prefer fixing OIDC.

### npm org

`ax` is unscoped but org-administered: scope and ownership are independent on
npm. The package's owners are the `ora-ai` org maintainers, and the
`ora-ai:developers` team holds a read-write grant
(`npm access grant read-write ora-ai:developers ax`), so org members publish to
it exactly as they do to `@ora-ai/*` packages.

## The rename from `@ora-ai/ax`

Versions ≤ `0.5.3` shipped as `@ora-ai/ax`; the plain `ax` name was acquired in
August 2026 and everything from the first `ax` release onward ships there. Two
consequences:

- `ax@0.0.1`–`0.2.2` predate us — an unrelated 2011 logging library that came
  with the name. All are deprecated and must never be reused; every release must
  version above them (the `0.5.x` line already does).
- `@ora-ai/ax` stays published but deprecated, its message pointing here. Don't
  publish to it again.

## When something goes wrong

**The release PR is not opening.** release-please only proposes a release when
at least one `feat:` or `fix:` commit landed since the last tag. Check the
`Release PR / tag` job log on the latest `main` push.

**The release PR merged but publish failed.** The tag and GitHub Release exist;
npm does not have the version yet. Fix the cause, then **Actions → Release →
Run workflow** with `dry_run` unchecked. The `publish` job reads the version
from `main`, confirms the tag exists, sees the version is not on npm, and
publishes. If it *is* already on npm the job logs a notice and exits green:
re-running is always safe.

**Publish succeeded but the workflow reported failure.** Re-run it; the
preflight finds the version on npm and skips. Nothing to fix by hand.

**Wrong version published.** You cannot reuse or overwrite a version. Land a
`fix:` and let the next release PR ship the correction forward. `npm deprecate`
the bad one with a message pointing at the replacement.

**The changelog or version in the release PR is wrong.** Do not edit the PR
branch; release-please force-pushes it. Fix the cause on `main` (a mistyped
commit title, a missing `Release-As:`) and the PR updates itself on the next
push.

## Optional hardening

Add a protected [environment](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments)
named `npm` with required reviewers, and set `environment: npm` on the
`publish` job. Publishing then needs a second person to approve the run. Worth
doing once more than one maintainer can merge release PRs. Update the trusted
publisher's environment name on npmjs.com in the same change.
