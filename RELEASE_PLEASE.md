# Centralized Release Please

This repository owns Burlington's two-stream release automation. The target repository has one
small `.github/workflows/release.yml` caller and two independent Release Please streams:
`workflow` and `release`.

The caller must reference an immutable central workflow release, for example:

```yaml
uses: divyasingh0/az_tf_workflow/.github/workflows/release-please.yml@v1.0.0
```

Do not reference `main` in production. Publish a new central workflow tag only after reviewing
the workflow, action version, permissions, and a test repository.

## Release flow

1. A developer opens a PR using the centrally standardized Release Notes template.
2. The PR uses a Conventional Commit title, such as `feat(storage): add lifecycle policy`.
3. The PR is merged to the configured target branch.
4. The caller invokes this reusable workflow on the target-branch push.
5. `googleapis/release-please-action` analyzes Conventional Commits and creates or updates a
   Release PR. No custom version, tag, or release Bash is used.
6. Merging the Release PR updates the changelog and version state.
7. Release Please creates the semantic version tag and GitHub Release.

Release Please determines the semantic version from Conventional Commits:

| Commit | Version |
|---|---|
| `fix(component): ...` | patch |
| `feat(component): ...` | minor |
| `feat(component)!: ...` or `BREAKING CHANGE:` | major |

## Repository release streams

The central workflow always uses Release Please manifest mode with exactly two packages:
`workflow` and `release`. Component ownership is path-based: a component change must modify files
under its configured directory. The Conventional Commit scope must use the same component name:

```text
feat(storage): add lifecycle policy
```

Release Please's manifest package selection is based on configured package paths, not custom
scope-parsing Bash. Keep component paths isolated and reject a PR that uses a component scope while
changing another component's files through normal review or policy checks.

The [`examples/local-two-components/`](./examples/local-two-components/) fixture matches this
setup. Both streams start at `2.0.0`.

Expected test results:

| Commit | Release PR | Tag |
|---|---|---|
| `feat(workflow): add reusable validation` | workflow only | `workflow-v2.1.0` |
| `fix(release): correct release metadata` | release only | `release-v2.0.1` |
| `feat(release)!: change release contract` | release only | `release-v3.0.0` |

## Testing the single repository with `workflow` and `release` tags

Release Please creates GitHub PRs, tags, and releases through the GitHub API, so a completely
offline local run cannot produce the real result. Use a temporary GitHub repository for the
authoritative test.

1. Create one temporary GitHub repository. Do not create separate repositories for `workflow` and
   `release`.
2. Copy the contents of `examples/local-two-components/` into its root. The resulting repository
   should look like:

   ```text
   .github/workflows/release.yml
   .release-please-config.json
   .release-please-manifest.json
   workflow/
   release/
   ```

3. In `.github/workflows/release.yml`, temporarily change:

   ```yaml
   uses: divyasingh0/az_tf_workflow/.github/workflows/release-please.yml@v1.0.0
   ```

   to the central workflow branch or test tag containing your change.
4. Push the repository to GitHub, enable Actions, and allow the workflow to write contents and
   pull requests.
5. Commit and push a change only under `workflow/`:

   ```text
   feat(workflow): add reusable validation
   ```

6. Confirm the Release Please PR only targets `workflow`.
7. Merge that Release PR and confirm:

   ```text
   workflow-v2.1.0
   ```

   The `release` version must remain `2.0.0`.
8. Commit and push a change only under `release/`:

   ```text
   fix(release): correct release metadata
   ```

9. Confirm the Release Please PR only targets `release`. Merge it and confirm:

   ```text
   release-v2.0.1
   ```

   The existing `workflow-v2.1.0` tag must remain unchanged.
10. Test a breaking release:

   ```text
   feat(release)!: change release contract
   ```

   Expected tag:

   ```text
   release-v3.0.0
   ```

At the end, the one repository should contain tags similar to:

```text
workflow-v2.1.0
release-v2.0.1
release-v3.0.0
```

Do not use `workflow-v` and `release-v` as two manually configured tag prefixes. The official
Release Please implementation uses manifest packages and `include-component-in-tag` to create
these independent tags.

For a local Actions simulation, install [`act`](https://github.com/nektos/act), provide a
fine-grained test token with repository contents and pull-request access, and run:

```text
act push \
  -W examples/local-two-components/.github/workflows/release.yml \
  -s GITHUB_TOKEN="$(gh auth token)"
```

`act` validates workflow wiring, but the temporary GitHub repository is still required to verify
the actual Release Please PR, tag, and GitHub Release lifecycle.

## Release Notes

The central `.github/PULL_REQUEST_TEMPLATE.md` is the canonical template. Burlington
organization/repository settings should distribute that template to consumer repositories so
developers do not maintain divergent copies. The final Release Please changelog contains the
Conventional Commit-derived release entries and the PR links; the PR itself contains the
business/technical context from the standardized Release Notes section.

Required sections are:

- Summary
- Changes
- Affected Component
- Impact
- Breaking Changes
- Migration / Upgrade Notes
- Validation

## Security and permissions

The reusable workflow grants only `contents: write` and `pull-requests: write`, scoped again at
the job level. It does not request `id-token`, issues, deployments, or package permissions.
`GITHUB_TOKEN` is used by default. A centrally managed GitHub App/PAT may be passed as the optional
`RELEASE_PLEASE_TOKEN` secret when branch protection or organization policy requires it.

## Validation before publishing a central workflow version

```text
actionlint
yamllint .github/workflows/release-please.yml
jq empty examples/local-two-components/.release-please-config.json
jq empty examples/local-two-components/.release-please-manifest.json
```

Exercise the two-stream caller from a test repository with:

- a `fix` commit and expected patch release;
- a `feat` commit and expected minor release;
- a breaking-change commit and expected major release;
- unrelated component changes in manifest mode;
- repeated runs to verify Release PR updates are idempotent;
- Release PR merge to verify the component tag and GitHub Release.

The action version is intentionally explicit (`v4.2.0`). Review the official
[Release Please Action](https://github.com/googleapis/release-please-action) release notes before
upgrading it.
