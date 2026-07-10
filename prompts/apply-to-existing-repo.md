# Prompt: Apply Verified Installer Setup To An Existing Repo

```text
Use the `setup-verified-installer` skill from:

https://github.com/danielraffel/verified-release-skills/tree/main/setup-verified-installer

I want to set up this existing repository so releases are easy to install and
easy to verify.

Path to Repo: [Add local path or GitHub URL]

Use this skill when the repo has, or should have, an installer path. If the same
repo also publishes `.pkg`, `.dmg`, `.zip`, `.tar.gz`, or other downloadable
release artifacts, still use this skill; it covers both the installer and the
release artifacts.

My goals:

- Keep the simple installer flow, like `curl ... | sh` or the project's
  equivalent.
- Add checksum verification for the installer and all user-facing release
  binaries.
- Publish source-controlled checksum manifests for install scripts.
- Publish release `SHA256SUMS` for the binaries the release workflow creates.
- Use GitHub Release asset digests as an additional verification source where
  available.
- Verify downloaded archives before extracting or executing them.
- Make README verification clear, but collapse longer verification details in a
  `<details>` section so onboarding still flows.
- Configure signed commits and signed tags so GitHub shows Verified where
  appropriate.
- Configure signing for both my personal identity and the automation or bot
  identity if this repo needs automated releases.
- Follow GitHub's current signing-key rules. Prefer Ed25519 SSH signing keys
  where appropriate, upload them as signing keys, and make sure committer
  emails are verified.
- Use 1Password as the backup for signing keys. For unattended automation on a
  secure machine, use a local file-backed signing key only where needed so
  builds and releases do not block on me approving 1Password prompts.
- Do not leak private keys in logs, shell history, commits, or workflow output.

Please inspect the repo first and identify:

- How releases are built today.
- Which binaries and assets users actually install.
- Whether there is already an installer, upgrade command, package manager flow,
  or second-stage download.
- Which README and docs pages need updates.
- Whether docs are generated or mirrored somewhere else.
- Which GitHub Actions workflows need changes.
- Whether artifact attestations and immutable releases should be enabled.

Implement the smallest complete version that makes this real.

Validation I expect before calling it done:

- Run the README install and verification commands from a clean temporary
  directory.
- Prove checksum mismatch or tamper failure is caught.
- Test both latest-release and pinned-version install paths if both exist.
- Verify release assets with `SHA256SUMS`.
- Verify GitHub Release asset digests where available.
- Run `gh release verify` and `gh release verify-asset`, or the repo's
  equivalent if available.
- Verify artifact attestations if enabled.
- Verify signed commits and signed tags locally and via GitHub API or UI status
  where possible.
- Confirm no private key material appears in the repo, logs, or generated docs.
- Run the repo's normal tests and docs gates.
- Leave unrelated dirty files alone.

When finished, summarize exactly what changed, what was verified, what remains
manual, and what I should screenshot for a blog post.
```
