# Centralized Release Please

This repository owns Burlington's release automation. Consumer repositories have one small
`.github/workflows/release.yml` caller and keep their Release Please configuration/state in
`.release-please-config.json` and, for manifest repositories, `.release-please-manifest.json`.

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

## Single-application repository

Use `release_mode: single`, configure `package_name`, and keep the initial version at `2.0.0`.
The example produces `edna-v2.0.0`, `edna-v2.1.0`, and so on because the consumer config enables
`include-component-in-tag` and `tag-separator: "-"`.

## Multi-component repository

Use `release_mode: manifest`. Each package key in `.release-please-config.json` is a component
directory and has an entry in `.release-please-manifest.json`. The central workflow always enables
`separate-pull-requests: true`.

For the manifest model, component ownership is path-based: a component change must modify files
under that component's configured directory. The Conventional Commit scope must use the same
component name and is used for human review and changelog context:

```text
feat(storage): add lifecycle policy
```

With the example layout, this produces `storage-v2.1.0` without changing unrelated component
versions. Release Please's manifest package selection is based on configured package paths, not
custom scope-parsing Bash. Therefore, repositories must keep component paths isolated and should
reject a PR that uses a component scope while changing another component's files through normal
review or policy checks.

### Two-stream local test fixture

The [`examples/local-two-components/`](./examples/local-two-components/) fixture models one
repository containing two independent components: `workflow` and `release`. Both start at
`2.0.0`.

Expected test results:

| Commit | Release PR | Tag |
|---|---|---|
| `feat(workflow): add reusable validation` | workflow only | `workflow-v2.1.0` |
| `fix(release): correct release metadata` | release only | `release-v2.0.1` |
| `feat(release)!: change release contract` | release only | `release-v3.0.0` |

## Testing locally

Release Please creates GitHub PRs, tags, and releases through the GitHub API, so a completely
offline local run cannot produce the real result. Use a temporary GitHub repository for the
authoritative test.

1. Copy the contents of `examples/local-two-components/` into a temporary repository.
2. Change the caller's reusable-workflow reference from `@v1.0.0` to the branch or tag containing
   your local central-workflow change.
3. Push the repository to GitHub and enable Actions.
4. Commit and push:

   ```text
   feat(workflow): add reusable validation
   ```

5. Confirm the Release Please PR only targets `workflow`.
6. Merge that Release PR and confirm `workflow-v2.1.0`.
7. Commit and push:

   ```text
   fix(release): correct release metadata
   ```

8. Confirm the Release Please PR only targets `release`, then merge it and confirm
   `release-v2.0.1`.

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
jq empty examples/single-application/.release-please-config.json
jq empty examples/multi-component/.release-please-config.json
jq empty examples/multi-component/.release-please-manifest.json
```

Exercise both example callers from a test repository with:

- a `fix` commit and expected patch release;
- a `feat` commit and expected minor release;
- a breaking-change commit and expected major release;
- unrelated component changes in manifest mode;
- repeated runs to verify Release PR updates are idempotent;
- Release PR merge to verify the component tag and GitHub Release.

The action version is intentionally explicit (`v4.2.0`). Review the official
[Release Please Action](https://github.com/googleapis/release-please-action) release notes before
upgrading it.
