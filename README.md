# Verified Installer Skill

This repo contains a Codex skill for setting up installer and release
verification in GitHub-hosted projects.

The skill is here:

https://github.com/danielraffel/verified-installer-skill/tree/main/setup-verified-installer

It is meant for projects that already publish, or are about to publish, release
binaries and want a more complete trust path:

- a simple `curl | sh` installer for convenience;
- a no-pipe verification path for people who want to inspect first;
- `SHA256SUMS` for install scripts and release assets;
- GitHub Release asset digest checks where useful;
- artifact attestations for release outputs;
- signed commits and tags that GitHub can mark as Verified;
- 1Password-backed recovery for signing keys;
- unattended file-backed signing keys for automation when human approval would
  block releases.

## Use It

Point an agent at the skill folder and the target repo:

```text
Use the `setup-verified-installer` skill from:

https://github.com/danielraffel/verified-installer-skill/tree/main/setup-verified-installer

I want to set up this existing repository so releases are easy to install and
easy to verify.

Path to Repo: [Add local path or GitHub URL]
```

The longer reusable prompt lives in
[prompts/apply-to-existing-repo.md](prompts/apply-to-existing-repo.md).

## Repo Layout

```text
setup-verified-installer/
  SKILL.md
  agents/openai.yaml
prompts/
  apply-to-existing-repo.md
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
