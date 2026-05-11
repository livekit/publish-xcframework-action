# publish-xcframework-action

Reusable GitHub Action that publishes an XCFramework zip to a hosting repo via **draft release + PR**.

Takes a pre-built `.xcframework.zip` and a set of already-rendered files (Package.swift, podspecs, sources, …) and:

1. Creates a **draft release** in the hosting repo with the zip attached
2. Copies the provided files to a **release branch** in the hosting repo
3. Opens a **PR** for review

Rendering of `Package.swift` / podspec templates is the caller's responsibility — render them in your own workflow with whatever engine you like (`sed`, `envsubst`, tera, stencil, a build script) and hand the finished files to this action.

## Usage

```yaml
- name: Compute checksum
  id: cs
  run: echo "checksum=$(shasum -a 256 ./build/MyFramework.xcframework.zip | awk '{print $1}')" >> "$GITHUB_OUTPUT"

- name: Render Package.swift
  run: |
    sed -e "s|{{VERSION}}|${VERSION}|g" \
        -e "s|{{CHECKSUM}}|${{ steps.cs.outputs.checksum }}|g" \
        templates/Package.swift > /tmp/Package.swift

- uses: livekit/publish-xcframework-action@v1
  with:
    xcframework-zip: ./build/MyFramework.xcframework.zip
    version: ${{ env.VERSION }}
    hosting-repo: livekit/my-xcframework
    files: |
      /tmp/Package.swift:Package.swift
      /tmp/MyFramework.podspec:MyFramework.podspec
    token: ${{ secrets.HOSTING_REPO_PAT }}
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `xcframework-zip` | yes | — | Path to the `.xcframework.zip` file |
| `version` | yes | — | Version tag (e.g., `0.0.5`) |
| `hosting-repo` | yes | — | Target repo in `owner/name` format |
| `files` | yes | — | Newline-separated `source:destination` pairs, copied verbatim |
| `token` | yes | — | GitHub PAT with `contents: write` and `pull_requests: write` on the hosting repo |
| `branch-prefix` | no | `release/` | Prefix for release branch name |
| `commit-author-name` | no | `github-actions[bot]` | Git author name |
| `commit-author-email` | no | `41898282+github-actions[bot]@users.noreply.github.com` | Git author email |
| `pr-body` | no | `""` | Extra text appended to the PR body |
| `release-body` | no | `""` | Extra text appended to the draft release notes |

Both the PR and draft release default to a body that links back to the source commit that produced the build:

```
Built by [`livekit/webrtc-build@abc1234`](https://github.com/livekit/webrtc-build/commit/abc1234...)
```

The PR body additionally includes a `> [!IMPORTANT]` reminder to merge before publishing the release. `pr-body` and `release-body` are appended below this default.

## Outputs

| Output | Description |
|--------|-------------|
| `checksum` | SHA256 of the xcframework zip (computed by the action) |
| `release-id` | GitHub release ID |
| `pr-number` | PR number |
| `pr-url` | PR URL |
| `branch` | Created branch name |

The action computes `checksum` for display in the PR/release body and as an output, but it is not used to render any files — callers must compute it themselves to fill in their templates.

## Workflow After the Action Runs

1. **Review the PR** in the hosting repo
2. **Merge the PR** — Package.swift with rendered checksum lands on the default branch
3. **Publish the draft release** — creates the version tag and makes the zip downloadable

Order matters: **merge first, then publish**. This ensures the tag points to a commit with the correct Package.swift.

## Examples

### Simple — render with sed

```yaml
- name: Compute checksum
  id: cs
  run: echo "checksum=$(shasum -a 256 build/LiveKitWebRTC.xcframework.zip | awk '{print $1}')" >> "$GITHUB_OUTPUT"

- name: Render templates
  run: |
    mkdir -p out
    for f in Package.swift LiveKitWebRTC.podspec; do
      sed -e "s|{{VERSION}}|${VERSION}|g" \
          -e "s|{{CHECKSUM}}|${{ steps.cs.outputs.checksum }}|g" \
          ".github/templates/$f" > "out/$f"
    done

- uses: livekit/publish-xcframework-action@v1
  with:
    xcframework-zip: ./build/LiveKitWebRTC.xcframework.zip
    version: ${{ env.VERSION }}
    hosting-repo: livekit/webrtc-xcframework
    files: |
      out/Package.swift:Package.swift
      out/LiveKitWebRTC.podspec:LiveKitWebRTC.podspec
    token: ${{ secrets.SWIFT_PUBLISH_PAT }}
```

### Complex — multiple files + bindgen sources

```yaml
- name: Build xcframework and render Package.swift
  run: cargo make swift-package

- uses: livekit/publish-xcframework-action@v1
  with:
    xcframework-zip: ./packages/swift/LiveKitUniFFI/RustLiveKitUniFFI.xcframework.zip
    version: ${{ env.VERSION }}
    hosting-repo: livekit/livekit-uniffi-xcframework
    files: |
      packages/swift/LiveKitUniFFI/Package.swift:Package.swift
      packages/swift/LiveKitUniFFI/Package@swift-6.2.swift:Package@swift-6.2.swift
      packages/swift/LiveKitUniFFI/LiveKitUniFFI.podspec:LiveKitUniFFI.podspec
      packages/swift/LiveKitUniFFI/Sources/LiveKitUniFFI/livekit_uniffi.swift:Sources/LiveKitUniFFI/livekit_uniffi.swift
    token: ${{ secrets.SWIFT_PUBLISH_PAT }}
```

## Token Permissions

The fine-grained PAT needs these permissions on the hosting repo:

- **Contents**: Read and write
- **Pull requests**: Read and write
- **Metadata**: Read (granted by default)

The hosting repo must be **public** for SPM binary target downloads to work (SPM cannot authenticate asset downloads).

## Testing

The repo includes a test workflow at `.github/workflows/test.yml` that can be triggered manually:

1. Add a `TEST_HOSTING_PAT` secret to the action repo (PAT with write access to your test hosting repo)
2. Run the workflow via **Actions > Test publish-xcframework > Run workflow**
3. Provide your test hosting repo (e.g., `myorg/test-xcframework-hosting`)
