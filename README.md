# Burlington Release Automation

Centralized GitHub Actions workflows for semantic versioning and GitHub Releases.

This repository provides reusable release automation for multiple repositories.
Each consuming repository configures its own default stream and may use any
additional Conventional Commit scopes as independent release streams.

Examples include `workflow-v` and `release-v`, or `module-v` and `network-v`.

## Architecture

The release process has three workflows:

| Workflow | Purpose |
|---|---|
| `.github/workflows/release-workflow.yml` | Trigger entry point and release-stream selection |
| `.github/workflows/version.yml` | Calculates the next semantic version |
| `.github/workflows/release.yml` | Creates the annotated tag and GitHub Release |

The entry-point workflow should be the only workflow attached to `push` and
`workflow_dispatch` events. The other workflows are reusable workflows.

## Release-stream selection

### Default stream

The `RELEASE_DEFAULT_STREAM` repository variable defines the default stream.
If it is `workflow`, these commit messages select `workflow-v`:

```text
feat: add workflow validation
fix: correct workflow behavior
chore: update workflow configuration
feat(workflow): add reusable workflow support
feat(workflow)!: change the workflow contract
```

### Explicit scoped streams

Any valid Conventional Commit scope selects a matching `<scope>-v` stream:

```text
feat(release): add release automation
fix(network): correct network metadata
feat(network)!: change the network contract
feat!(provider): change the provider contract
```

For normal push events, the merged PR title is inspected first. If the title
contains a scope, the matching `<scope>-v` stream is selected. Unscoped
commits use the configured default stream.

For a manual workflow dispatch, the `stream` input can explicitly select:

```text
default
release
network
```

Avoid mixing different streams in the same PR. Use separate PRs so every
release has one clear version stream.

## Conventional Commits

Commit messages determine whether a release is created and which semantic
version increment is applied.

| Commit message | Version change |
|---|---|
| `fix(scope): ...` | Patch |
| `feat(scope): ...` | Minor |
| `feat(scope)!: ...` | Major |
| `feat!(scope): ...` | Major, supported for compatibility |
| Commit body contains `BREAKING CHANGE:` | Major |
| `docs: ...`, `test: ...`, `chore: ...` | No release unless manually overridden |

Recommended syntax:

```text
feat(workflow): add reusable validation
fix(release): correct release metadata
feat(release)!: change release contract
```

The `!` belongs after the scope:

```text
feat(workflow)!: breaking change
```

The alternate form is also supported:

```text
feat!(workflow): breaking change
```

## Version examples

Assume the latest tag is `workflow-v3.0.0`.

```text
feat(workflow): add feature
```

Creates:

```text
workflow-v3.1.0
```

```text
fix(workflow): correct bug
```

Creates:

```text
workflow-v3.0.1
```

```text
feat(workflow)!: change interface
```

Creates:

```text
workflow-v4.0.0
```

The `release` stream is calculated independently:

```text
fix(release): correct release metadata
```

Creates the next `release-v` patch tag without changing the `workflow-v`
version.

## Pull Request description format

Every releasable PR must contain this section:

```markdown
## Release Notes

### Summary

<!-- Required: one or two sentences describing the business or technical outcome. -->

### Changes

<!-- Required: describe the externally relevant changes. Do not paste commit history. -->

### Impact

- [ ] No impact
- [ ] Low
- [ ] Medium
- [ ] High

### Breaking Changes

- [ ] No
- [ ] Yes

**Explanation (required when Yes):**

<!-- Explain the breaking change and affected consumers. -->
```

The canonical template is:

```text
.github/pull_request_template.md
```

Select exactly one Impact option and exactly one Breaking Changes option.
Summary and Changes must contain meaningful text.

## Published GitHub Release notes

The release workflow publishes only the `## Release Notes` section from the
merged PR. Other PR content and routine commit history are excluded.

The workflow normalizes the selected values:

```markdown
### Impact

Low
```

If no breaking change is selected:

```markdown
### Breaking Changes

No
```

If a breaking change is selected:

```markdown
### Breaking Changes

Yes

#### Explanation

The public module interface changed.
```

The workflow also adds:

```markdown
### Approved By

reviewer-login
```

Approved reviewers are read from GitHub PR review metadata. If no approved
review is recorded, the PR merger is used as a fallback. This is preferable to
asking developers to manually enter approval names.

The GitHub Release title uses the last commit title from the merged PR, for
example:

```text
feat(workflow): add reusable validation
```

## Release flow

1. Create a branch.
2. Make the change.
3. Use a Conventional Commit message.
4. Open a PR using the standard Release Notes template.
5. Select exactly one Impact value.
6. Select exactly one Breaking Changes value.
7. Add an explanation when Breaking Changes is `Yes`.
8. Obtain review and merge the PR into `main`.
9. The entry workflow identifies the release stream.
10. The version workflow finds the latest tag for that stream.
11. The version workflow calculates the semantic version.
12. The release workflow creates an annotated tag.
13. The release workflow publishes the normalized PR Release Notes.

## Testing examples

### Workflow stream

```bash
git commit -m "feat(workflow): add reusable validation"
git push origin main
```

Expected result:

```text
workflow-v2.1.0
```

### Release stream

```bash
git commit -m "fix(release): correct release metadata"
git push origin main
```

Expected result:

```text
release-v2.0.1
```

### Breaking change

```bash
git commit -m "feat(release)!: change release contract"
git push origin main
```

Expected result:

```text
release-v3.0.0
```

### Manual dispatch

Use GitHub Actions **Run workflow** and select:

```text
stream: workflow
```

or:

```text
stream: release
```

Use `dry_run: true` to calculate and display the release without creating a
tag or GitHub Release.

## Required repository permissions

The workflow requires:

```yaml
permissions:
  contents: write
  pull-requests: read
```

The repository Actions settings must allow `GITHUB_TOKEN` to write repository
contents and create GitHub Releases. Pull requests should require the standard
Release Notes section through repository governance or review policy.

## Initial versions and tags

The default initial version is `2.0.0`, configurable through the repository
variable:

```text
RELEASE_INITIAL_VERSION=2.0.0
```

The workflow searches tags by prefix:

```text
<default-stream>-v*
<scoped-stream>-v*
```

Each prefix has an independent version history.

Configure the default stream per repository:

```text
RELEASE_DEFAULT_STREAM=workflow
```

Other repositories can use values such as `module`, `application`, or
`infrastructure`.

## Troubleshooting

### No releasable commits found

Check that the commit follows Conventional Commits:

```text
feat(workflow): add feature
fix(release): correct issue
```

For a breaking change, use:

```text
feat(workflow)!: breaking change
```

Ensure the selected scope matches the intended stream.

### Wrong stream selected

For a merged PR, the workflow uses the PR title. Use:

```text
feat(network): network-specific change
```

If no scope is present, the configured `RELEASE_DEFAULT_STREAM` is used.

### Release notes validation failed

Confirm that the merged PR contains:

- `## Release Notes`
- Non-empty `### Summary`
- Non-empty `### Changes`
- Exactly one checked Impact option
- Exactly one checked Breaking Changes option
- Explanation when Breaking Changes is `Yes`

### Node.js deprecation warning

A GitHub Actions Node.js runtime warning is not a release-calculation failure.
The workflow can continue unless the action itself fails.

## Operational guidance

- Use separate PRs for separate release streams.
- Do not manually create or move release tags.
- Do not edit the calculated version in workflow scripts.
- Keep the default stream as `workflow` unless governance changes it.
- Review tag history before using a new stream.
- Protect `main` and require PR approval.
- Keep the Release Notes template centrally managed.
- Test workflow changes in a non-production repository first.
