# Verified Release Skills

This repo contains Codex skills for setting up release verification in
GitHub-hosted projects.

There are two related skills:

- [setup-verified-installer](setup-verified-installer) is for projects that
  need a `curl | sh` or PowerShell installer.
- [setup-verified-release-artifacts](setup-verified-release-artifacts) is for
  projects that publish downloadable release artifacts, such as `.pkg`, `.dmg`,
  `.zip`, `.tar.gz`, `.exe`, `.msi`, `.deb`, `.rpm`, `.AppImage`, `.apk`, or
  `.aab`, but do not need an install script.

They are separate skills because installer hardening and release-artifact
provenance have different failure modes, but they share the same trust model:

- `SHA256SUMS` for user-facing files;
- GitHub Release asset digest checks where useful;
- artifact attestations for release outputs;
- signed commits and tags that GitHub can mark as Verified;
- immutable GitHub Releases when the release flow is draft-first;
- 1Password-backed recovery for signing keys;
- unattended file-backed signing keys for automation when human approval would
  block releases.

## Choose A Skill

Use `setup-verified-installer` when the repo has, or should have, an installer:

```text
curl -fsSL https://example.com/install.sh | sh
```

That skill covers the installer script, no-pipe verification, release archives,
second-stage downloads, upgrade commands, and verifying downloads before
extraction.

Use `setup-verified-release-artifacts` when the repo only publishes files people
download from GitHub Releases:

```text
Project-1.2.3.pkg
Project-1.2.3.dmg
Project-1.2.3-win64.zip
Project-1.2.3-linux-x64.tar.gz
```

That skill covers checksums, GitHub asset digests, artifact attestations when
the workflow handles the final bytes, immutable releases, platform-specific
verification commands, and concise README instructions.

## Use It

For an installer repo, point an agent at the installer skill:

```text
Use the `setup-verified-installer` skill from:

https://github.com/danielraffel/verified-release-skills/tree/main/setup-verified-installer

I want to set up this existing repository so releases are easy to install and
easy to verify.

Path to Repo: [Add local path or GitHub URL]
```

For a release-artifact repo without an installer, use:

```text
Use the `setup-verified-release-artifacts` skill from:

https://github.com/danielraffel/verified-release-skills/tree/main/setup-verified-release-artifacts

I want to set up this existing repository so GitHub Release artifacts are easy
to verify, without adding an install script.

Repo path:
<ABSOLUTE_REPO_PATH>

GitHub repo:
<OWNER>/<REPO>
```

Longer reusable prompts live in:

- [prompts/apply-to-existing-repo.md](prompts/apply-to-existing-repo.md)
- [prompts/apply-release-artifacts-to-existing-repo.md](prompts/apply-release-artifacts-to-existing-repo.md)

## Tested Example Prompt

This prompt was tested against
[`danielraffel/pulp-example-plugins`](https://github.com/danielraffel/pulp-example-plugins),
which publishes a signed and notarized macOS `.pkg` release asset without an
installer script:

```text
Use the `setup-verified-release-artifacts` skill from:

https://github.com/danielraffel/verified-release-skills/tree/main/setup-verified-release-artifacts

I want to set up danielraffel/pulp-example-plugins so GitHub Release artifacts
are easy to verify, without adding an install script.

Inspect the repo first, classify how releases are built today, and add the
smallest complete setup for SHA256SUMS, GitHub Release asset digest
verification, artifact attestations where honest, immutable-release readiness,
and concise README verification instructions.

Do not overclaim provenance if artifacts are built manually or outside GitHub
Actions. Leave unrelated dirty files alone. Validate the README commands from a
clean temp directory before calling it done.
```

## Repo Layout

```text
setup-verified-installer/
  SKILL.md
  agents/openai.yaml
setup-verified-release-artifacts/
  SKILL.md
  agents/openai.yaml
prompts/
  apply-to-existing-repo.md
  apply-release-artifacts-to-existing-repo.md
```

## Sources

- GitHub Release asset digests:
  https://github.blog/changelog/2025-06-03-releases-now-expose-digests-for-release-assets/
- GitHub release integrity verification:
  https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/verify-release-integrity
- GitHub commit signature verification:
  https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification
- GitHub artifact attestations:
  https://docs.github.com/en/actions/concepts/security/artifact-attestations
- Deno installer verification example:
  https://github.com/denoland/deno_install
