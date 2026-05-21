---
name: npm-trusted-publishing
description: Use when hardening npm package release workflows with trusted publishing, OIDC, GitHub environments, pinned GitHub Actions, disabled publish-path caching, Changesets release PRs, or direct tag-based npm publish flows.
---

# npm Trusted Publishing

## Core Policy

Prefer npm trusted publishing over long-lived `NPM_TOKEN` publishes. The safe shape is:

- npm package trusts one exact GitHub Actions workflow, and usually the `npm` environment.
- The publish job has `permissions: id-token: write`; release-PR/build jobs do not.
- The publish job is gated by a GitHub environment with required reviewers and branch restrictions.
- Publish-path dependency caching is disabled.
- Every external GitHub Action in the publish workflow is pinned to a full commit SHA with the original tag kept as a comment.
- npm package settings require 2FA and disallow token publishes.
- npm trusted publisher allowed actions are narrowed to `npm stage publish` only when the workflow uses staged publishing.

Do not treat the workflow PR as complete until the human/package-owner has configured the npm and GitHub-side trust controls.

## First Pass Audit

1. Find publish workflows:
   - `.github/workflows/release.yml`, `publish.yml`, or any workflow containing `npm publish`, `changeset publish`, `changesets/action`, `NPM_TOKEN`, or `id-token`.
2. Identify the flow:
   - **Changesets flow**: release PR on `main`, then publish after the release PR is merged.
   - **Direct flow**: package artifact is built, validated, and published from a release tag.
3. Check package metadata:
   - `publishConfig.provenance: true` is good but not sufficient.
   - Remove secret-token assumptions from publish jobs.
4. Check trust boundary:
   - Only the final npm publish job gets `id-token: write`.
   - Stable publish uses `environment: npm`.
   - Canary/prerelease jobs are not silently moved behind the stable environment unless asked.

## Changesets Flow

Use this for repos like `jovidecroock/pracht`.

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
      - run: npm install -g npm@11.15.0
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
      - run: npm install -g npm@11.15.0
      - run: pnpm build
      - uses: changesets/action@<sha> # vX
        with:
          publish: node .github/scripts/stage-packages.mjs
          commitMode: github-api
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Add an unpublished-package check when a merge to `main` can happen without a publishable version. In monorepos, skip fixture/example packages and the workspace root unless it is intentionally published.

## Direct Tag Publish Flow

Use this for repos like `preactjs/preact`, where a tag builds an artifact and publishes the tarball.

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
      - run: npm install -g npm@11.15.0
      - name: Validate package
        run: |
          # Verify tarball package name and version match the tag before publishing.
      - name: Determine dist-tag
        run: |
          # Stable tags publish as latest; prereleases only use approved tags.
      - run: npm stage publish <package>.tgz --provenance --access public --tag "${{ steps.dist-tag.outputs.tag }}"
```

Keep the build/test job separate from the OIDC publish job. If the publish job installs dependencies, disable setup-node's package-manager cache there too.

### Mirroring Direct Publish Changes to Maintenance Branches

For repos with active release branches, mirror staged-publishing workflow changes onto each branch that can still publish.

Example: `preactjs/preact` needed the same release workflow change on both `main` and `v10.x`:

1. Inspect the source PR and exact workflow diff:
   ```bash
   gh pr view <pr> --repo <owner>/<repo> --json title,baseRefName,headRefName,commits,files,url,state
   git fetch origin main v10.x pull/<pr>/head:pr-<pr> --prune
   git show --stat --patch <source-sha> -- .github/workflows/release.yml
   ```
2. Create a branch from the maintenance branch and cherry-pick the workflow commit:
   ```bash
   git checkout -B <topic>-v10 origin/v10.x
   git cherry-pick <source-sha>
   ```
3. Verify the maintenance-branch workflow contains the exact staged-publish shape:
   ```bash
   python3 - <<'PY'
   from pathlib import Path
   s = Path('.github/workflows/release.yml').read_text()
   assert 'npm install -g npm@11.15.0' in s
   assert 'npm stage publish' in s
   PY
   ```
4. Push and open a PR against the maintenance branch, not the default branch:
   ```bash
   git push -u origin HEAD
   gh pr create --base v10.x --head <branch> --title "Use npm staged publishing" --body-file /tmp/pr.md
   ```

Do not assume release workflow hardening on `main` protects older published lines. If `v10.x`, `v9.x`, or another branch can tag and publish independently, mirror the change or explicitly document why it is out of support.

## Staged Publishing

Use staged publishing when CI should upload release artifacts but a maintainer should still approve the public release with 2FA.

- Requires npm CLI `11.15.0` or later and Node `22.14.0` or later.
- Replace direct `npm publish` with `npm stage publish` in trusted publish jobs.
- Configure the npm trusted publisher's **Allowed actions** to allow `npm stage publish` and disallow `npm publish`; otherwise OIDC can still publish directly and bypass the staged approval gate.
- For Changesets repos, use a small repository script such as `.github/scripts/stage-packages.mjs` to find unpublished package versions and call `npm stage publish <package-dir> --provenance --access public --tag <tag>`.
- Capture and surface all stage IDs in CI logs; maintainers approve after review with `npm stage approve <stage-id>` or on npmjs.com.
- Do not put public-release side effects in package `postpublish` hooks; those run during staging, not final approval.

## Disable Publish-Path Caching

In any job that can mint npm OIDC credentials or run `npm publish`/`changeset publish`:

- Set `package-manager-cache: false` on every `actions/setup-node` step.
- Remove `cache: npm`, `cache: pnpm`, and `cache-dependency-path` from setup-node in that job.
- Remove `actions/cache` restores for `~/.npm`, pnpm stores, Yarn caches, or build outputs consumed by publish.
- Prefer fresh install + build inside the gated publish job, or publish a validated artifact built by a separate job.

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

## Required npm Configuration Prompt

After changing the workflow, prompt the package owner to configure npm. Trusted publishing is not active from GitHub YAML alone.

Ask them to do one of these. For staged-publishing workflows, the trust relationship should be stage-only:

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
- Workflow filename: `<workflow.yml>`
- Environment: `npm`
- Allowed actions: enable **npm stage publish** and disable **npm publish** for staged-publishing workflows

Also ask them to set package publishing access to:

- **Require two-factor authentication and disallow tokens**

This is currently a package setting in npmjs.com. It prevents granular access tokens from publishing even if they were created with 2FA bypass.

## GitHub Environment and Branch Protection

Create or verify a GitHub environment named `npm`:

```bash
OWNER_REPO=<owner>/<repo>
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
```

Then verify it:

```bash
gh api "repos/$OWNER_REPO/environments/npm" --jq '{name, protection_rules, deployment_branch_policy}'
```

Also configure branch protection on the default branch so release workflow edits require review. Prefer preserving existing required status checks and review settings. Do not overwrite mature branch protection with a minimal config unless the repo had no protection.

For an unprotected `main`, seed a minimal protection policy:

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

## PR Checklist

Include this in the PR body:

- [ ] Publish job uses npm trusted publishing/OIDC, not `NPM_TOKEN`.
- [ ] Only the publish job has `id-token: write`.
- [ ] Publish job is gated by `environment: npm`.
- [ ] Publish-path caching is disabled.
- [ ] External actions in the trusted publish workflow are pinned to SHAs.
- [ ] npm trusted publisher configured for `<owner>/<repo>` + `<workflow.yml>` + `npm` environment.
- [ ] npm trusted publisher allowed actions are stage-only: `npm stage publish` enabled, `npm publish` disabled.
- [ ] npm package publishing access set to require 2FA and disallow tokens.
- [ ] GitHub `npm` environment has required reviewers and protected-branch deployment policy.
- [ ] Default branch protection requires PR review for workflow/config changes.

## Common Mistakes

- Leaving `id-token: write` on the Changesets release-PR job.
- Assuming `publishConfig.provenance` enables trusted publishing. It does not configure npm's trust policy.
- Keeping setup-node caching because the install is slow. The publish path should optimize for supply-chain safety, not speed.
- Pinning `actions/checkout` but forgetting `changesets/action`, `pnpm/action-setup`, `actions/github-script`, or `actions/download-artifact`.
- Configuring npm trust without the same environment name used in the workflow.
- Leaving `npm publish` enabled in the trusted publisher allowed actions after switching CI to `npm stage publish`; OIDC could still publish directly if a workflow regresses.
- Running `npm trust` once in a workspace and assuming it configured every package; the command is package-oriented and currently unaware of workspaces, so configure each published package explicitly.
- Using self-hosted GitHub runners for npm trusted publishing; npm trusted publishing requires supported hosted CI runners.
