# Prompt: Apply Verified Release Artifact Setup To An Existing Repo

```text
Use the `setup-verified-release-artifacts` skill from:

https://github.com/danielraffel/verified-release-skills/tree/main/setup-verified-release-artifacts

I want to set up this existing repository so GitHub Release artifacts are easy
to verify, without adding an install script.

Repo path:
<ABSOLUTE_REPO_PATH>

GitHub repo:
<OWNER>/<REPO>

My goals:

- Support the release artifacts this repo publishes today, such as `.pkg`.
- Keep the setup ready for future macOS, Windows, Linux, Android, or iOS
  release artifacts if the repo adds them later.
- Publish release `SHA256SUMS` covering every user-facing downloadable asset.
- Use GitHub Release asset digests as an additional verification source where
  available.
- Add GitHub artifact attestations only when the final release bytes are
  produced by, or honestly handled by, a GitHub Actions workflow.
- Do not claim build provenance for assets that are built entirely outside
  GitHub Actions.
- Enable immutable releases only if the release flow can publish draft-first:
  create draft, upload every asset, upload `SHA256SUMS`, verify, then publish.
- Add concise README verification instructions near the download instructions.
  Collapse longer explanations if they would interrupt onboarding.
- Include platform-specific checks only when they match the asset type, such as
  `pkgutil` or `spctl` for macOS `.pkg`, `Get-FileHash` for Windows, and
  `sha256sum --check` for Linux.
- Configure signed commits or signed tags separately if source provenance is in
  scope for this repo.
- Do not leak private keys in logs, shell history, commits, or workflow output.

Please inspect the repo first and identify:

- How releases are built and uploaded today.
- Which release assets users actually download.
- Whether the release is Actions-built, hybrid, manual/local-built, or
  source-only.
- Whether `SHA256SUMS` already exists on recent releases.
- Whether latest release assets expose GitHub `digest` fields.
- Whether immutable releases are enabled.
- Whether release tags are signed.
- Which README and docs pages need verification instructions.
- Which GitHub Actions workflows need changes, if any.

Implement the smallest complete version that makes this real.

Validation I expect before calling it done:

- Run the README verification commands from a clean temporary directory.
- Download release assets and verify them with `SHA256SUMS`.
- Verify GitHub Release asset digests where available.
- Verify artifact attestations if enabled.
- Run `gh release verify` and `gh release verify-asset` if immutable releases
  are enabled and supported for this repo.
- Verify platform-specific signatures or package checks for the asset types the
  repo publishes.
- Confirm the docs do not overclaim provenance.
- Run the repo's normal tests and docs gates.
- Leave unrelated dirty files alone.

When finished, summarize exactly what changed, what was verified, what remains
manual, and what should be screenshotted for a blog post or release notes.
```
