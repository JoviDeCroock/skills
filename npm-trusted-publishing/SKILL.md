---
name: npm-trusted-publishing
description: Use when hardening npm package release workflows with trusted publishing, OIDC, GitHub environments, pinned GitHub Actions, disabled publish-path caching, Changesets release PRs, direct tag-based npm publish flows, staged publishing, or npm release-supply-chain reviews.
version: 1.0.0
license: MIT
metadata:
  hermes:
    tags: [npm, trusted-publishing, oidc, github-actions, staged-publishing, supply-chain]
    related_skills: [github-actions-publishing, github-pr-workflow]
---

# npm Trusted Publishing

## Overview

Prefer npm trusted publishing over long-lived `NPM_TOKEN`/`NODE_AUTH_TOKEN` publishes. Trusted publishing lets npm accept publishes from one configured CI workflow identity via OIDC instead of a reusable secret.

This skill is intentionally security-biased, but do not apply every hardening step blindly. Separate what is required for npm trusted publishing from optional release-process hardening, and call out anything the package owner must configure outside the pull request.

The goal is to reduce risk areas, if a push to main can just publish then a compromised SSH key or GitHub account is enough to introduce risk to the ecosystem.

People can use https://drydock.org to do security scanning on staged publishes or configure custom GitHub environment gates.

## When to Use

Use this when:

- Migrating npm publishing from tokens to trusted publishing/OIDC.
- Reviewing release workflows that run `npm publish`, `npm stage publish`, `changeset publish`, `changesets/action`, semantic-release, or custom publish scripts.
- Adding GitHub environments, branch restrictions, or CODEOWNERS around npm release jobs.
- Hardening publish jobs by removing caches and pinning release-path GitHub Actions.
- Designing Changesets release-PR or direct tag-based publish flows.
- Adding staged publishing so CI stages packages and a maintainer approves with 2FA.

Do not use this as the only guide for:

- Non-npm registries.
- General CI speed or test workflow cleanup.
- Publishing from unsupported/self-hosted runners unless npm docs say the provider/runner is supported.
- Repos where the owner explicitly wants low-friction token publishing and accepts that risk.

## Requirements Matrix

| Capability | Required versions / conditions |
|---|---|
| npm trusted publishing | npm CLI `>=11.5.1`, Node `>=22.14.0`, supported hosted CI runner, configured npm trusted publisher, `id-token: write` on the publish job |
| npm staged publishing | npm CLI `>=11.15.0`, Node `>=22.14.0`, `npm stage publish`, maintainer approval/rejection with 2FA |
| pnpm staged publishing | npm CLI `>=11.3.0`, Node `>=22.14.0`, `pnpm stage publish`, maintainer approval/rejection with 2FA |
| GitHub Actions trusted publisher | GitHub-hosted runners only unless npm docs say otherwise; configure the workflow filename and optional environment on npmjs.com |
| Provenance | Generated automatically by npm when using trusted publishing; private source repositories may not get public provenance |

Use `npm@11.15.0` or newer when the workflow stages packages. For direct trusted publishing, `npm@11.5.1` or newer is sufficient, but using the current npm 11 is usually simpler.

## Core Policy

The safe shape is:

- npm trusts one exact GitHub Actions workflow filename, usually with the `npm` environment.
- Only the final publish/stage job has `permissions: id-token: write`; release-PR, test, and build jobs do not.
- The publish job is gated by a GitHub environment with required reviewers and branch/deployment restrictions when the repo can support it.
- Publish-path dependency/build caching is disabled, or the job publishes a verified artifact produced by a trusted job in the same run.
- External GitHub Actions in the publish workflow are pinned to full commit SHAs, with the original tag kept as a comment.
- npm package settings require 2FA and disallow token publishes when possible.
- For staged publishing, the npm trusted publisher's allowed actions enable `npm stage publish` and disable `npm publish`.

Do not treat a workflow PR as complete until the package owner has configured npm-side trust and reviewed GitHub-side environment controls. GitHub YAML alone does not enable trusted publishing.

## First-Pass Audit

1. Find publish workflows:
   - Check `.github/workflows/release.yml`, `.github/workflows/publish.yml`, and any workflow containing `npm publish`, `npm stage publish`, `changeset publish`, `changesets/action`, `semantic-release`, `NPM_TOKEN`, `NODE_AUTH_TOKEN`, or `id-token`.
2. Identify the release flow:
   - **Changesets flow**: release PR on `main`, then publish/stage after the release PR is merged.
   - **Direct tag flow**: package artifact is built, validated, and published/staged from a release tag.
   - **semantic-release flow**: semantic-release calculates the version and publishes; apply its npm auth caveats.
3. Check package metadata:
   - `publishConfig.provenance: true` is good defense-in-depth but does not configure npm's trust policy.
   - With trusted publishing, npm provenance is automatic; do not rely on `--provenance` as the core mechanism.
4. Check trust boundary:
   - Only the final npm publish/stage job gets `id-token: write`.
   - Stable publishing uses `environment: npm` when possible.
   - Canary/prerelease jobs are not silently moved behind the stable environment unless asked.
5. Check triggers:
   - Do not publish from `pull_request` or `pull_request_target`.
   - Be cautious with `workflow_run`; it can accidentally trust artifacts from untrusted workflows.
   - For `workflow_dispatch`, require an environment approval or equivalent maintainer gate.
   - For tag flows, verify tag name, package version, tarball version, and dist-tag before publishing.

Quick local scan:

```bash
grep -R "NPM_TOKEN\|NODE_AUTH_TOKEN\|npm publish\|npm stage publish\|id-token\|changesets/action\|semantic-release" \
  .github/workflows package.json .npmrc 2>/dev/null || true
```

## Changesets Flow

Use this for repos where Changesets opens a release PR and publishes after the release PR is merged.

```yaml
jobs:
  release:
    outputs:
      has_changesets: ${{ steps.changesets.outputs.hasChangesets }}
      should_publish: ${{ steps.publish-check.outputs.should_publish }}
    permissions:
      contents: write
      issues: read
      pull-requests: write
    steps:
      - uses: actions/checkout@<sha> # vX
        with:
          persist-credentials: false
      - uses: pnpm/action-setup@<sha> # vX
      - uses: actions/setup-node@<sha> # vX
        with:
          node-version: 22
          registry-url: https://registry.npmjs.org
          package-manager-cache: false
      - run: pnpm install --frozen-lockfile --ignore-scripts
      - run: npm install -g npm@^11.15.0
      - id: changesets
        uses: changesets/action@<sha> # vX
        with:
          version: pnpm run version
          commitMode: github-api
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      - id: publish-check
        if: steps.changesets.outputs.hasChangesets == 'false'
        run: node .github/scripts/has-unpublished-packages.mjs

  publish:
    needs: release
    if: needs.release.outputs.should_publish == 'true'
    concurrency:
      group: npm-publish-${{ github.ref }}
      cancel-in-progress: false
    environment:
      name: npm
      url: https://www.npmjs.com/package/<package-or-org>
    permissions:
      contents: write
      id-token: write
    steps:
      - uses: actions/checkout@<sha> # vX
        with:
          persist-credentials: false
      - uses: pnpm/action-setup@<sha> # vX
      - uses: actions/setup-node@<sha> # vX
        with:
          node-version: 22
          registry-url: https://registry.npmjs.org
          package-manager-cache: false
      - run: pnpm install --frozen-lockfile --ignore-scripts
      - run: npm install -g npm@^11.15.0
      - run: pnpm build
      - uses: changesets/action@<sha> # vX
        with:
          publish: node .github/scripts/stage-packages.mjs
          commitMode: github-api
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Add an unpublished-package check when a merge to `main` can happen without a publishable version. In monorepos, skip fixture/example packages, private packages, and the workspace root unless it is intentionally published.

`npm stage`/`pnpm stage` is unaware of workspaces. Scripts such as `.github/scripts/stage-packages.mjs` must explicitly iterate package directories and stage each published package separately.

## Direct Tag Publish Flow

Use this when a tag builds an artifact and the publish job publishes/stages that tarball.

```yaml
permissions:
  contents: read

jobs:
  build:
    permissions:
      contents: read
    uses: ./.github/workflows/build-test.yml

  release:
    needs: build
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@<sha> # vX
      - uses: actions/download-artifact@<sha> # vX
      - uses: actions/github-script@<sha> # vX

  publish:
    needs: [build, release]
    concurrency:
      group: npm-publish-${{ github.ref }}
      cancel-in-progress: false
    environment:
      name: npm
      url: https://www.npmjs.com/package/<package>
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/download-artifact@<sha> # vX
      - uses: actions/setup-node@<sha> # vX
        with:
          node-version: 24
          registry-url: https://registry.npmjs.org
          package-manager-cache: false
      - run: npm install -g npm@^11.15.0
      - name: Validate package
        run: |
          # Verify tarball package name and version match the tag.
          tar -xOf <package>.tgz package/package.json | node -e '
          let p = JSON.parse(require("fs").readFileSync(0, "utf8"))
          console.log(`${p.name}@${p.version}`)
          '
      - name: Determine dist-tag
        run: |
          # Stable tags publish as latest; prereleases only use approved prerelease tags.
      - run: npm stage publish <package>.tgz --access public --tag "${{ steps.dist-tag.outputs.tag }}"
```

Keep the build/test job separate from the OIDC publish job. If the publish job installs dependencies, disable setup-node's package-manager cache there too.

If publishing a downloaded artifact:

- Publish only artifacts produced by the same workflow run.
- Verify artifact checksum before publish when a checksum is available.
- Inspect tarball contents before publish/stage.
- Verify package name/version from `package/package.json` inside the tarball.
- Verify the tag name and dist-tag are correct, and never publish prereleases to `latest` unless explicitly intended.

## Semantic-Release Caveats

semantic-release can use npm trusted publishing, but do not blindly copy direct `npm publish` setup:

- Grant `id-token: write` only to the semantic-release job that publishes.
- Configure the npm trusted publisher for the workflow that triggers the release.
- Do not blindly set `registry-url` in `actions/setup-node` when semantic-release handles npm auth; setup-node writes an `.npmrc` and can cause auth/token errors in semantic-release flows.
- Follow semantic-release's current trusted publishing recipe when semantic-release owns the publish command.

## Reusable Workflow Caveat

If publish logic lives in a reusable workflow called via `workflow_call`, configure npm trusted publishing for the caller workflow that receives the `push`, `workflow_dispatch`, or tag event. npm authorizes the initiating workflow identity, not merely the file containing the publish command.

In the PR body, name the exact workflow filename the package owner must configure on npmjs.com.

## Staged Publishing

Use staged publishing when CI should upload release artifacts but a maintainer should still approve the public release with 2FA.

- Requires npm CLI `11.15.0` or later and Node `22.14.0` or later.
- Replace direct `npm publish` with `npm stage publish` in trusted publish jobs.
- Configure the npm trusted publisher's **Allowed actions** to allow `npm stage publish` and disallow `npm publish`; otherwise OIDC can still publish directly and bypass the staged approval gate.
- For Changesets repos, use a small repository script such as `.github/scripts/stage-packages.mjs` to find unpublished package versions and call `npm stage publish <package-dir> --access public --tag <tag>`.
- Capture and surface all stage IDs in CI logs; maintainers approve after review with `npm stage approve <stage-id>` or on npmjs.com.
- Do not put public-release side effects in package `postpublish` hooks; those run during staging, not final approval.
- Tags are immutable for staged packages. To change a staged package's tag, reject it and stage again.

## Provenance Guidance

With trusted publishing, npm generates provenance automatically. The workflow usually does not need `--provenance`.

Keep `publishConfig.provenance: true` if already present, but do not claim it enables trusted publishing. Trusted publishing is enabled by npm-side trust configuration plus CI OIDC permissions.

Caveats:

- Private source repositories may not get public npm provenance even when trusted publishing authenticates correctly.
- If supporting an older/non-trusted publish path, `--provenance` may still be relevant; call that out explicitly.
- Do not disable provenance unless there is a documented compatibility reason.

## Disable Publish-Path Caching

In any job that can mint npm OIDC credentials or run `npm publish`, `npm stage publish`, or `changeset publish`:

- Set `package-manager-cache: false` on every `actions/setup-node` step.
- Remove `cache: npm`, `cache: pnpm`, and `cache-dependency-path` from setup-node in that job.
- Remove `actions/cache` restores for `~/.npm`, pnpm stores, Yarn caches, or build outputs consumed by publish.
- Prefer fresh install + build inside the gated publish job, or publish a validated artifact built by a separate job.

The goal is to prevent attacker-controlled cached dependency/build output from entering a credentialed publish path. Caching is acceptable in non-credentialed test/build jobs if the publish job either rebuilds from source or verifies the artifact digest produced by a trusted job.

Package-manager notes:

- npm: use `npm ci`; avoid npm cache restores in publish jobs.
- pnpm: use `pnpm install --frozen-lockfile --ignore-scripts`; avoid pnpm store cache restores in publish jobs.
- Yarn: use immutable installs; avoid cache restores in publish jobs.
- If relying on `packageManager`, enable Corepack explicitly and pin the package-manager version.

## Pin External Actions

Pin all external `uses:` entries in the trusted publish workflow:

```yaml
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```

Rules:

- Pin GitHub-hosted third-party and first-party actions to 40-character SHAs.
- Keep the tag as a trailing comment for maintainability.
- Skip local actions (`./.github/actions/...`) and non-GitHub specs such as `docker://...`.
- Resolve tags with `gh api repos/<owner>/<repo>/git/ref/tags/<tag>` or `git ls-remote --tags`.

Tradeoff: SHA pinning improves release-path integrity but adds maintenance burden. Prioritize workflows with publish credentials/OIDC. If the repo uses Renovate or Dependabot, configure it to update GitHub Actions SHAs and preserve tag comments.

## Required npm Configuration Prompt

After changing the workflow, prompt the package owner to configure npm. Trusted publishing is not active from GitHub YAML alone.

For staged-publishing workflows, the trust relationship should be stage-only:

```bash
npm install -g npm@^11.15.0
npm trust github --repo <owner>/<repo> --file <workflow.yml> --env npm --allow-stage-publish --no-allow-publish <package>
npm trust list <package>
```

If a repo intentionally still publishes directly from CI, use `--allow-publish` instead; do not enable both unless there is a concrete transition reason and the PR calls that out.

Or configure it in npmjs.com:

- Package → Settings → Trusted publishing
- Provider: GitHub Actions
- Repository: `<owner>/<repo>`
- Workflow filename: `<workflow.yml>`; filename only, not `.github/workflows/<file>`
- Environment: `npm`, if the workflow uses `environment: npm`
- Allowed actions: enable **npm stage publish** and disable **npm publish** for staged-publishing workflows

Also ask them to set package publishing access to:

- **Require two-factor authentication and disallow tokens**

This is currently a package setting in npmjs.com. It prevents granular access tokens from publishing even if they were created with 2FA bypass.

For monorepos, configure the trusted publisher separately for each published package. Do not assume one `npm trust` invocation covers every workspace package.

## GitHub Environment and Branch Protection

Prefer inspect-before-mutate. Environment and branch protection are side-effectful admin settings.

Inspect current state first:

```bash
OWNER_REPO=<owner>/<repo>
gh api "repos/$OWNER_REPO/environments/npm" --jq '{name, protection_rules, deployment_branch_policy}' || true
gh api "repos/$OWNER_REPO/branches/main/protection" || true
```

If the user/package owner wants the agent to manage GitHub settings, create or verify a GitHub environment named `npm`:

```bash
REVIEWER=<github-user>
USER_ID=$(gh api "users/$REVIEWER" --jq .id)

gh api --method PUT "repos/$OWNER_REPO/environments/npm" --input - <<JSON
{
  "prevent_self_review": true,
  "reviewers": [{ "type": "User", "id": $USER_ID }],
  "deployment_branch_policy": {
    "protected_branches": true,
    "custom_branch_policies": false
  }
}
JSON

gh api "repos/$OWNER_REPO/environments/npm" --jq '{name, protection_rules, deployment_branch_policy}'
```

Also configure branch protection on the default branch so release workflow edits require review. Preserve existing required status checks and review settings. Do not overwrite mature branch protection with a minimal config unless the repo had no protection and the owner asked for a seed policy.

For an unprotected `main`, a minimal seed policy is:

```bash
gh api --method PUT "repos/$OWNER_REPO/branches/main/protection" --input - <<'JSON'
{
  "required_status_checks": null,
  "enforce_admins": false,
  "required_pull_request_reviews": {
    "required_approving_review_count": 1
  },
  "restrictions": null,
  "required_linear_history": false,
  "allow_force_pushes": false,
  "allow_deletions": false,
  "block_creations": false,
  "required_conversation_resolution": false,
  "lock_branch": false,
  "allow_fork_syncing": true
}
JSON
```

For workflow/config review ownership, add `.github/CODEOWNERS` when the repo lacks it:

```text
# Request review for GitHub configuration changes.
.github/ @<reviewer>
```

If you lack permissions to configure npm or GitHub admin settings, put exact owner actions in the PR body rather than pretending the migration is complete.

## Verification Commands

Use these before declaring the workflow side done:

```bash
# Check for lingering token assumptions.
grep -R "NPM_TOKEN\|NODE_AUTH_TOKEN\|//registry.npmjs.org.*_authToken" .github/workflows package.json .npmrc 2>/dev/null || true

# Check publish/OIDC locations.
grep -R "npm publish\|npm stage publish\|changeset publish\|id-token" .github/workflows 2>/dev/null || true

# Verify configured npm trust for each package, if authenticated as a package maintainer.
npm trust list <package>

# Verify GitHub environment, if using one.
gh api "repos/<owner>/<repo>/environments/npm" --jq '{name, protection_rules, deployment_branch_policy}'

# Verify workflow YAML as GitHub sees it.
gh workflow view <workflow.yml> --repo <owner>/<repo> --yaml
```

For an already-published version, verify npm state with commands such as:

```bash
npm view <package>@<version> name version dist.integrity dist.tarball
```

Check provenance/attestation visibility on npmjs.com or with `npm view` where supported; do not hardcode a field unless the current npm CLI supports it.

## PR Checklist

Split the checklist by who can verify it.

Agent-verifiable:

- [ ] Publish/stage job uses npm trusted publishing/OIDC, not `NPM_TOKEN` or `NODE_AUTH_TOKEN`.
- [ ] Only the publish/stage job has `id-token: write`.
- [ ] Publish/stage job is gated by `environment: npm` when the flow uses an npm environment.
- [ ] Publish-path caching is disabled or artifact integrity is verified before publish/stage.
- [ ] External actions in the trusted publish workflow are pinned to SHAs.
- [ ] Unsafe triggers (`pull_request`, `pull_request_target`, untrusted `workflow_run`) cannot publish.
- [ ] Tag/package/tarball validation exists for direct tag flows.
- [ ] Reusable workflow/caller workflow identity is documented when relevant.

Human/package-owner-verifiable:

- [ ] npm trusted publisher configured for `<owner>/<repo>` + `<workflow.yml>` + `npm` environment, if used.
- [ ] For staged publishing, npm trusted publisher allowed actions are stage-only: `npm stage publish` enabled, `npm publish` disabled.
- [ ] npm package publishing access set to require 2FA and disallow tokens.
- [ ] GitHub `npm` environment has required reviewers and protected-branch deployment policy.
- [ ] Default branch protection requires PR review for workflow/config changes.
- [ ] Every published package in a monorepo has its own trusted publisher configuration.

## Common Mistakes

- Leaving `id-token: write` on the Changesets release-PR, test, or build job.
- Assuming `publishConfig.provenance` enables trusted publishing. It does not configure npm's trust policy.
- Adding `--provenance` everywhere and missing that trusted publishing now generates provenance automatically.
- Keeping setup-node caching because install is slow. The publish path should optimize for supply-chain safety, not speed.
- Pinning `actions/checkout` but forgetting `changesets/action`, `pnpm/action-setup`, `actions/github-script`, `actions/download-artifact`, or semantic-release-related actions.
- Configuring npm trust without the same environment name used in the workflow.
- Configuring the reusable workflow file instead of the caller workflow file on npmjs.com.
- Leaving `npm publish` enabled in the trusted publisher allowed actions after switching CI to `npm stage publish`; OIDC could still publish directly if a workflow regresses.
- Running `npm trust` once in a workspace and assuming it configured every package; the command is package-oriented and `npm stage` is currently workspace-unaware.
- Using self-hosted GitHub runners for npm trusted publishing when npm only supports GitHub-hosted runners.
- Publishing from `pull_request`, `pull_request_target`, or untrusted `workflow_run` triggers.
- Publishing a tarball without verifying its name/version/tag and contents.
