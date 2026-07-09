---
name: setup-verified-installer
description: >-
  Set up a generic curl/PowerShell installer with checksum verification,
  README verification instructions, release-workflow checksum publishing,
  artifact attestations, and signed source refs.
  Use when a user says they want to make a project easy to install, add a
  curl | sh installer, harden an install.sh/install.ps1 flow, publish
  SHA256SUMS, verify release assets before extraction, use GitHub Release asset
  digests, add GitHub artifact attestations, configure Verified commits/tags,
  configure unattended file-backed SSH signing keys with 1Password backup,
  configure 1Password-agent signing, back signing keys up to 1Password, or
  capture a Deno/Helm/Syft-style installer verification pattern.
---

# Setup Verified Installer

## Overview

Create a boring, copy-pasteable installer path that keeps the simple one-liner
while giving cautious users a no-pipe checksum path and making every downloaded
release archive verify before extraction.

The complete trust stack has separate layers. Keep them separate in docs and
tests:

- signed commits/tags identify the source ref as coming from a configured human
  or bot identity;
- build attestations connect release assets to the workflow and source ref that
  produced them;
- immutable GitHub Releases plus release asset digests identify the published
  bytes;
- project-owned `SHA256SUMS` gives users a plain `curl` plus
  `shasum`/`sha256sum` verification path;
- installer internals verify every downloaded archive before extraction.

The target shape is:

```text
convenience:
  curl -fsSL https://example.com/install.sh | sh

cautious:
  curl -fLso install.sh https://example.com/install.sh
  curl -fsSL https://raw.githubusercontent.com/ORG/REPO/main/path/SHA256SUMS \
    | shasum -a 256 -c --ignore-missing
  less install.sh
  sh install.sh

installer internals:
  detect platform -> resolve release -> download archive -> verify hash
  -> extract/install -> run post-install checks

release provenance:
  signed commit/tag -> build workflow -> artifact attestation
  -> immutable release + GitHub asset digest + SHA256SUMS
```

## Workflow

1. Inspect the repo before designing the installer.
   - Search for current install surfaces: `README`, docs, `install.sh`,
     `install.ps1`, release workflows, upgrade commands, SDK/plugin/cache
     downloaders, and any existing checksum helpers.
   - Identify docs ownership before editing prose: README-only, generated docs,
     website mirrors, getting-started guides, CLI help, package-manager pages,
     or docs-source templates. Update the source of truth and run the repo's
     docs checks when the install flow appears in more than one place.
   - Identify the release host, asset names, version tag convention, and whether
     releases are draft-then-publish or uploaded after publication.
   - Find second-stage downloads. If the installer downloads a CLI and then the
     CLI downloads an SDK, cache, plugin, or update, cover both stages.
   - Audit source-ref signing separately: are commits signed, are release tags
     signed, and does automation create tags with a human key, bot key, or no
     signature?

2. Keep the simple install, but do not pretend it is intrinsically safe.
   - Keep the one-line `curl | sh` or `irm | iex` path for convenience.
   - Put a no-pipe verification block directly below it in the README.
   - Explain that checksum verification protects the bytes being downloaded; it
     does not make executing an unreviewed network script risk-free.

3. Publish two checksum surfaces when applicable.
   - Add an installer-script checksum manifest in source control, usually near
     the installer, with basename entries:

     ```text
     <64 hex chars>  install.sh
     <64 hex chars>  install.ps1
     ```

   - Publish release `SHA256SUMS` as a release asset covering every
     user-facing archive. Generate it after all release assets are uploaded but
     before an immutable release is published.

4. Use GitHub Release asset digests as an additional source, not as the only
   manual path.
   - GitHub release assets expose `digest` values such as `sha256:<hex>` through
     release metadata. Use that when the installer or native CLI already parses
     release metadata.
   - Still publish `SHA256SUMS` because it works with plain `curl` plus
     `shasum`/`sha256sum`, without `gh`, `jq`, Homebrew, cosign, or GPG.
   - When both GitHub's digest and `SHA256SUMS` are available, require them to
     agree. Fail closed on disagreement.
   - Do not add `jq` as a shell-installer prerequisite just to parse JSON. For
     POSIX shell, `SHA256SUMS` can be the required verifier; native CLIs and
     PowerShell can parse release metadata more reliably.

5. Verify before extraction everywhere.
   - Resolve the exact release tag and asset name.
   - For `latest`, resolve release metadata once and use that tag, asset URL,
     and digest for the rest of the install. Avoid races where `latest` changes
     between calls.
   - Download to a temporary directory.
   - Match checksum manifest entries by exact basename, not substring.
   - Require exactly one matching manifest entry.
   - Compute the local SHA-256.
   - Abort before `tar`, `unzip`, `Expand-Archive`, moving files into place, or
     executing downloaded binaries if the hash is missing or mismatched.
   - Print a concise success line such as `Verified SHA-256: <hash>`.

6. Define a legacy policy.
   - For releases created before checksums exist, do not silently skip
     verification.
   - Either reject the install with a clear message or require an explicit
     opt-in such as `--allow-unverified` or `PROJECT_ALLOW_UNVERIFIED=1`.

7. Add provenance after the checksum path is real.
   - Add artifact attestations to the workflow that produces the final release
     archive bytes.
   - Configure signed commits/tags as a separate key-management task, preferably
     proven in a disposable repo before changing the real release pipeline.
   - Do not create or print private keys as part of a normal code-editing
     session. Secret setup should be a narrow, auditable local operation.

## Release Workflow Pattern

Implement checksum publishing in the coordinator that runs after every platform
asset has been uploaded and before the release becomes immutable.

Enable immutable releases before creating the first release that should verify
with `gh release verify`. Immutability applies only to future releases:

```bash
gh api -X PUT repos/ORG/REPO/immutable-releases --silent
gh api repos/ORG/REPO/immutable-releases
```

For immutable releases, use a draft-first flow: create the draft, attach all
assets including `SHA256SUMS`, then publish the draft. Publishing creates the
release attestation used by `gh release verify`.

Use this checksum shape, adapted to the repo's release tooling:

```bash
gh release download "$TAG" --repo "$REPO" --dir release-assets
(
  cd release-assets
  rm -f SHA256SUMS checksums.sha256
  shasum -a 256 * | LC_ALL=C sort > SHA256SUMS
  shasum -a 256 -c SHA256SUMS
)
gh release upload "$TAG" release-assets/SHA256SUMS --repo "$REPO" --clobber
```

If the project already uses another checksum filename for automation, either
teach automation to use `SHA256SUMS` or upload the existing name as an alias.
Do not make users guess between names in docs.

After publication, verify both the release and a downloaded asset:

```bash
gh release verify "$TAG" --repo ORG/REPO
gh release download "$TAG" --repo ORG/REPO --dir verify-assets
gh release verify-asset "$TAG" verify-assets/project-archive.tar.gz --repo ORG/REPO
```

## Artifact Attestation Pattern

When a project builds release assets in GitHub Actions, add build provenance
attestations for the final archive files. This verifies "which workflow and
source ref produced this asset"; it does not replace checksum verification.
It is also separate from immutable release attestations: `gh attestation verify`
checks workflow provenance, while `gh release verify` checks the immutable
release record.

Use the current GitHub attestation action in the job that has the final asset
bytes:

```yaml
permissions:
  contents: read
  id-token: write
  attestations: write

steps:
  - name: Attest release artifacts
    uses: actions/attest@v4
    with:
      subject-path: |
        dist/project-*.tar.gz
        dist/project-*.zip
```

Document verification separately:

```bash
gh attestation verify "$asset" \
  -R ORG/REPO \
  --signer-workflow github.com/ORG/REPO/.github/workflows/release.yml
```

If a self-hosted runner is involved, say so honestly. The attestation gives
traceability to the workflow identity and source ref; it does not prove the
runner was uncompromised.

If `actions/attest` fails with `Feature not available for user-owned private
repositories`, make the disposable proof repo public or use an organization repo
where attestations are available. Do this during proofing before wiring a real
release pipeline.

## Signing Strategy

GitHub's `Verified` badge on commits and tags is useful source provenance, but
it answers a different question from release asset checksums. Treat signing as
a separate layer:

- personal commits/tags: use per-machine SSH signing keys;
- automation-created tags or commits: use a dedicated bot signing key;
- release assets: still require `SHA256SUMS`, GitHub release digests, and
  artifact attestations.

Choose the live signing path before creating keys:

- **Unattended local/Shipyard agents:** use file-backed OpenSSH private keys on
  disk with `0600` permissions, and store recovery copies in 1Password. This is
  the right path when Codex, Shipyard, CI runners, or headless scripts must sign
  without a human approving every operation. Prefer this by default on a secure
  developer machine when the user says agents should be able to sign while they
  are away.
- **Interactive human-only signing:** use the 1Password SSH agent and
  `op-ssh-sign` if the user prefers private keys to never leave 1Password.
  This can reduce prompts, but it is not truly unattended: the 1Password app
  must be installed, unlocked, and authorized for the process/session.

Prefer per-machine personal keys (`m1`, `m3`, `m5`, etc.) over one shared
personal key. A shared key is easier to copy but has a larger revocation blast
radius. Use a separate bot key for Shipyard or release automation; do not reuse
a personal key in CI.

### File-Backed Signing For Unattended Agents

This is the default path for "do not block on me" requests. Do not configure
`gpg.ssh.program` or `op-ssh-sign` for this mode.

Use Ed25519 keys. GitHub supports Ed25519 for SSH signing, while DSA is not
supported and RSA has extra SHA-2 constraints. Do not use ECDSA or DSA for this
workflow.

For a per-machine personal key:

```bash
machine=$(hostname -s)
email="$(git config user.email)"
key="$HOME/.ssh/github-signing-${machine}-daniel"

ssh-keygen -t ed25519 -N '' -C "github-signing:${machine}:${email}" -f "$key"
chmod 600 "$key"
chmod 644 "$key.pub"

git config --global gpg.format ssh
git config --global user.signingkey "$key"
git config --global --unset-all gpg.ssh.program || true
git config --global commit.gpgsign true
git config --global tag.gpgSign true
```

Audit SSH authentication separately from Git signing. Git may still fetch or
push over SSH, and a broad SSH config such as `Host * IdentityAgent
~/Library/Group Containers/2BUA8C4S2C.com.1password/t/agent.sock` can trigger
1Password prompts even when commit signing is file-backed. Check the effective
GitHub SSH config:

```bash
ssh -G github.com 2>/dev/null | grep -Ei 'identityagent|identityfile|identitiesonly|user '
```

If the user wants unattended GitHub operations and `identityagent` points at
1Password, add a scoped override before any broad `Host *` rule:

```sshconfig
Host github.com
  IdentityFile ~/.ssh/id_ed25519_macstudio
  IdentitiesOnly yes
  IdentityAgent none
```

Then prove SSH auth does not need an interactive prompt:

```bash
ssh -o BatchMode=yes -T git@github.com
```

Do not remove the user's global 1Password SSH agent setup if they use it for
other hosts. Scope the no-agent override to GitHub or to the specific automation
host that must run unattended.

Also check peer-machine aliases used by Shipyard, tartci, or local remote-build
flows. A broad `Host *` 1Password agent can still affect `m1`, `m3`, `m5`,
`blackbook`, `macstudio`, VM, Linux, or Windows aliases even after GitHub is
fixed. Add `IdentityAgent none` for those aliases too, while keeping their
existing `IdentityFile` entries:

```sshconfig
Host github.com m1 m3 m5 blackbook macstudio pulp-vm pulp-linux pulp-win dev 192.168.64.* 192.168.65.* 127.0.0.1 localhost
  IdentityAgent none
```

Check the exact hosts used by running automation, not only friendly aliases.
Shipyard/tartci/local-CI scripts may invoke raw targets such as
`admin@192.168.64.2`, `admin@192.168.64.3`, or localhost tunnel endpoints; a
`Host dev` override does not apply to a direct raw-IP connection. Inspect active
commands, then test the exact host:

```bash
ps auxww | grep -E 'ssh |shipyard|tartci' | grep -v grep
for host in 192.168.64.2 192.168.64.3 127.0.0.1 localhost; do
  ssh -G "$host" 2>/dev/null | grep -Ei '^(hostname|identityagent|identityfile|identitiesonly|user) '
done
```

If raw local VM hosts are part of unattended automation, add their subnet or
localhost patterns to the scoped no-agent block too. Tart VM IPs can rotate
between jobs, so prefer a bounded pattern over chasing one address at a time:

```sshconfig
Host dev 192.168.64.* 192.168.65.* 127.0.0.1 localhost
  IdentityAgent none
```

Prove every unattended alias resolves away from 1Password before running a
build or release workflow:

```bash
for host in github.com m1 m3 m5 blackbook dev 192.168.64.2 192.168.64.3 192.168.65.19 127.0.0.1 localhost; do
  ssh -G "$host" 2>/dev/null | grep -Ei '^(hostname|identityagent|identityfile|identitiesonly) '
done
```

Before relying on the key, make sure the committer email maps to the GitHub
account. GitHub's `no_user` verification reason means the signed commit's
committer email is not associated with any GitHub user, even if
`git verify-commit` succeeds locally. Check the effective config in the target
repo because repo-local config can override the global email:

```bash
git config --show-origin --get user.email
git config user.email verified-email@example.com
```

For a shared automation identity, create one bot key, copy it only to machines
or runners that need to sign, and use that key only for automation-created
commits/tags:

```bash
bot_key="$HOME/.ssh/github-signing-shipyard-bot"
ssh-keygen -t ed25519 -N '' -C "github-signing:shipyard-bot:ORG:$(date +%Y-%m-%d)" -f "$bot_key"
chmod 600 "$bot_key"
chmod 644 "$bot_key.pub"
```

Upload every public key to GitHub as a **signing** key, not just as an
authentication key. If using `gh`, this is:

```bash
gh ssh-key add "$key.pub" --type signing --title "GitHub file signing key - ${machine} - Daniel"
gh ssh-key add "$bot_key.pub" --type signing --title "GitHub file signing key - Shipyard bot"
```

Keep GitHub key titles explicit and prune obsolete duplicates. A good final
inventory looks like:

```text
GitHub file signing key - m1 - Daniel
GitHub file signing key - m3 - Daniel
GitHub file signing key - m5 - Daniel
GitHub file signing key - Shipyard bot
```

Configure local verification with an allowed signers file on every machine that
will verify signatures:

```bash
mkdir -p ~/.ssh
touch ~/.ssh/allowed_signers
chmod 700 ~/.ssh
chmod 600 ~/.ssh/allowed_signers
printf '%s %s\n' "$(git config user.email)" "$(cut -d' ' -f1,2 "$key.pub")" >> ~/.ssh/allowed_signers
git config --global gpg.ssh.allowedSignersFile ~/.ssh/allowed_signers
```

Then prove file-backed signing locally:

```bash
git commit --allow-empty -m "test: signed commit"
git verify-commit HEAD
git tag -s v0.0.0-signing-test -m "Signing test"
git tag -v v0.0.0-signing-test
```

Then prove GitHub agrees in a disposable repo or throwaway branch after pushing:

```bash
sha=$(git rev-parse HEAD)
gh api repos/ORG/REPO/commits/$sha \
  --jq '.commit.verification | {verified, reason, verified_at}'
```

The GitHub result must be `verified=true` and `reason=valid`. `unknown_key`
means the public key was not uploaded as a signing key. `no_user` means the
committer email on the commit is not associated with a GitHub account. Fix
those before using the setup in a production repo.

This proof must not show a 1Password approval prompt. If it does, Git is still
using the 1Password signer or agent instead of the file-backed key.

### 1Password-Agent Signing For Interactive Humans

Use this path only when interactive approval is acceptable. Required setup:

- install the 1Password desktop app;
- install the 1Password CLI (`op`);
- enable 1Password Developer settings for CLI integration;
- enable the 1Password SSH agent;
- install or generate the SSH key in 1Password;
- upload the public key to GitHub as a signing key.

Git config for this path:

```bash
git config --global gpg.format ssh
git config --global user.signingkey "<public ssh-ed25519 key>"
git config --global gpg.ssh.program /Applications/1Password.app/Contents/MacOS/op-ssh-sign
git config --global commit.gpgsign true
git config --global tag.gpgSign true
```

The 1Password prompt may offer "Approve for all applications". That is useful
for reducing repeated local prompts, but it is not a headless guarantee. The
authorization lasts only for the configured agent session/duration, and the
1Password app still needs to be unlocked for private-key use.

For a real release pipeline, prefer a disposable repo first: create a signed
commit/tag, publish a tiny release, upload `SHA256SUMS`, attest an asset, and
verify every layer before changing a production repo.

## Key Storage And 1Password Backup Pattern

Private signing keys are secrets. Never paste them into chat, logs, issue text,
or workflow output.

For unattended file-backed keys, 1Password is backup/escrow, not the live
signing path:

- store each private key in 1Password with a clear item name such as
  `GitHub file signing private key backup - m5 - Daniel`;
- store the bot key as `GitHub file signing private key backup - Shipyard bot`;
- include public key, fingerprint, creation date, live path, and GitHub signing
  key title/id in metadata or notes;
- tag items with `github-signing`, the project/org, `backup`, and
  `file-backed`;
- keep per-machine personal keys separate so one machine can be revoked without
  rotating every personal key.

The cleanest backup is to import the OpenSSH private key into a 1Password SSH
Key item using the desktop app. If using the CLI, verify the syntax locally
first. In practice, a Secure Note with a concealed `private key` field plus
visible `public key` and `fingerprint` fields is a reliable fallback for backup
when CLI SSH-key import support is unclear.

If the user asks for 1Password-backed recovery, verify 1Password prerequisites
up front:

- 1Password desktop app installed and unlocked;
- 1Password CLI (`op`) installed;
- Developer CLI integration enabled in 1Password settings;
- SSH agent enabled when using the interactive 1Password-agent signing path.

For automation:

- create a dedicated bot signing key;
- store it in 1Password under an automation/bot item;
- inject it into GitHub Actions or Shipyard only for the signing step;
- write it to a temporary file with restrictive permissions;
- configure Git with `user.signingkey` pointing at that private-key file for
  the signing step only;
- scrub the temp file in a trap/finally block;
- never print the private key or the full secret value.

If the 1Password CLI is available, first verify the local syntax before using
it:

```bash
op --version
op whoami
op item create --help
op item template get "Secure Note"
```

The `op` CLI is versioned and may not be installed on every machine. Do not
invent `op` commands from memory. If the CLI is unavailable or its syntax is not
clear, use the 1Password desktop app for the backup and record only the item
name in the task notes.

## README Pattern

Keep the first install block short:

```bash
curl -fsSL https://example.com/install.sh | sh
```

Then add a verification block. If the verification section is more than a few
commands, put the whole thing in a Markdown `<details>` block so cautious users
can expand it and everyone else can continue to the first real use of the tool:

````markdown
<details>
<summary><strong>Verification</strong> (optional checks, click to expand)</summary>

As an additional layer of security, download the shell installer and verify its
SHA-256 checksum before running it:

```bash
curl -fLso install.sh https://example.com/install.sh
curl -fsSL https://raw.githubusercontent.com/ORG/REPO/main/path/to/SHA256SUMS \
  | shasum -a 256 -c --ignore-missing
sh install.sh
```

Explain what each layer means here.

</details>
````

For shorter READMEs, an expanded verification block is also fine:

```bash
curl -fLso install.sh https://example.com/install.sh
curl -fsSL https://raw.githubusercontent.com/ORG/REPO/main/path/to/SHA256SUMS \
  | shasum -a 256 -c --ignore-missing
less install.sh
sh install.sh
```

After the collapsed section, resume with the next useful onboarding step, not
more trust-model prose. Good examples are `project create ...`, `tool init ...`,
or `tool --help`; optional editor/agent plugins should usually come after the
CLI is installed and the first project path is visible.

Treat the README as product onboarding, not just a checksum note. If the
verification block makes the opening flow feel heavy, suggest a clearer section
order, collapse explainers, or move detail to the docs. When the repo generates
or mirrors docs from another source, update that source instead of hand-editing
only the rendered README.

For Linux-heavy projects, also show the GNU form when useful:

```bash
curl -fsSL https://raw.githubusercontent.com/ORG/REPO/main/path/to/SHA256SUMS \
  | sha256sum --check --ignore-missing
```

If release archives are installable without the script, add a version-pinned
manual path:

```bash
version=1.2.3
asset=project-darwin-arm64.tar.gz
base="https://github.com/ORG/REPO/releases/download/v${version}"

curl -fsSLO "$base/$asset"
curl -fsSLO "$base/SHA256SUMS"
awk -v file="$asset" '$2 == file { print }' SHA256SUMS | shasum -a 256 -c -
tar xzf "$asset" -C "$HOME/.local/bin"
```

Use explicit version tags in cautious examples. `latest` is fine for
convenience, but it is not reproducible.

## Installer Script Contract

Keep the installer understandable:

- Detect OS and architecture.
- Resolve the requested version or `latest`.
- Select the exact asset name.
- Download to a temp directory with cleanup on exit.
- Verify SHA-256 before extraction.
- Extract into the install directory.
- Add/update PATH only after the binary is in place.
- Run a cheap post-install check such as `tool --version`.
- Trigger any second-stage downloads through the same verified path.

Avoid:

- Silent checksum fallback.
- Same-origin-only "binary plus checksum" claims when a stronger split source is
  easy.
- Grep patterns that can match the wrong asset.
- Replacing direct verification with Homebrew as the only answer.
- Requiring `gh`, Homebrew, cosign, GPG, or `jq` for the default installer.

## Validation Checklist

Before calling the setup done:

- Run the README verification command in a fresh temp directory.
- Verify the checksum manifest format with `shasum -a 256 -c`.
- Tamper with a downloaded archive and prove extraction does not run.
- Remove the checksum entry and prove install fails closed.
- Test explicit version and `latest` resolution.
- Test macOS/Linux shell paths and Windows PowerShell paths when both exist.
- Check second-stage downloads such as upgrade, SDK, plugin, or cache installs.
- Run repo formatting/docs checks such as `git diff --check`.
- Confirm release automation cannot publish without `SHA256SUMS`.
- Verify artifact attestations with `gh attestation verify` for every produced
  asset class.
- Verify signed commits and tags with `git verify-commit`, `git tag -v`, and
  GitHub's verification API or `Verified` badge. Treat `no_user` as a committer
  email configuration failure, not as a cryptographic-key failure.
- If GitHub remotes use SSH and unattended operation is required, verify
  `ssh -G github.com` does not route through the 1Password agent, then prove
  `ssh -o BatchMode=yes -T git@github.com` authenticates without a prompt.
- Prove automation can create a signed tag in a disposable repo before enabling
  signed release tags in a production repo.
- Confirm private signing keys are backed up in 1Password and never appear in
  shell history, workflow logs, or repository files.
