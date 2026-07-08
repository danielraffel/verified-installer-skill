# Making `install.sh` Easier To Trust

I like a simple installer.

```bash
curl -fsSL https://example.com/install.sh | sh
```

It is easy to paste. It works on a clean machine. It removes a lot of ceremony
for people who just want to try a tool.

It also asks people to execute a network script. That is a real trust request.
For Pulp, I wanted to keep the simple path, but also make the careful path much
clearer for people who care about where the code came from and whether the
bytes they downloaded are the bytes I published.

So I set up a repeatable pattern for this:

- keep the one-line installer;
- publish checksum files;
- verify downloaded archives before extraction;
- use GitHub's release asset digests;
- use GitHub release verification where it applies;
- sign commits and tags so GitHub can show Verified;
- back up signing keys in 1Password;
- turn the whole setup into a reusable Codex skill.

The skill is here:

https://github.com/danielraffel/verified-installer-skill/tree/main/setup-verified-installer

## The Short Version

There are three separate questions I wanted to answer.

First, did this source change come from a trusted identity?

That is what signed commits and tags help answer. GitHub can show a commit or
tag as Verified when it has a valid GPG, SSH, or S/MIME signature associated
with the right account. That does not prove the code is good. It says the source
ref was signed by a key GitHub knows about for that identity.

GitHub docs: https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification

Second, did this downloaded file change after it was published?

That is what SHA-256 checksums help answer. A checksum is a fingerprint of a
file. If the downloaded file has a different hash, it is not the same file. That
can catch accidental corruption, a bad mirror, the wrong asset, or tampering
between publishing and installation.

Third, which workflow built this release artifact?

That is what artifact attestations help answer. An attestation can link a
release artifact back to the GitHub Actions workflow and source ref that
produced it. It still does not prove the software is secure. It gives the person
installing it more provenance to inspect.

GitHub docs: https://docs.github.com/en/actions/concepts/security/artifact-attestations

## What Changed Recently

GitHub now exposes SHA-256 digests for release assets. GitHub announced this on
June 3, 2025. The digest is generated when the asset is uploaded and is visible
in the Releases UI, the REST API, GraphQL, and the GitHub CLI.

GitHub changelog: https://github.blog/changelog/2025-06-03-releases-now-expose-digests-for-release-assets/

That is useful because an installer can compare the downloaded archive with the
digest GitHub reports for that asset.

I still like publishing `SHA256SUMS` too. It is boring, portable, and works with
plain `curl` plus `shasum` or `sha256sum`. Nobody needs Homebrew, `jq`, GPG,
cosign, or the GitHub CLI just to verify a file.

This is close to what Deno does for its installer verification path:

https://github.com/denoland/deno_install

## The README Shape

For Pulp, I wanted the README to stay useful for both kinds of users.

The top stays simple:

```bash
curl -fsSL https://www.generouscorp.com/pulp/install.sh | sh
```

Then the verification path is available, but collapsed:

```bash
curl -fLso install.sh https://www.generouscorp.com/pulp/install.sh
curl -fsSL https://raw.githubusercontent.com/danielraffel/pulp/main/tools/install/SHA256SUMS \
  | shasum -a 256 -c --ignore-missing
sh install.sh
```

That lets careful users download the script, verify it, inspect it, and then run
it. Everyone else can keep moving to the next useful step.

The order matters. I do not want the first page of a project to become a
security whitepaper. I want the trust details to be available without burying
the command that gets someone to a working install.

## What The Installer Should Do

The installer itself should verify every archive before extraction.

The flow I want is:

```text
detect platform
resolve version
select exact asset
download to a temp directory
verify SHA-256
extract
install
run a cheap post-install check
```

The important part is that verification happens before `tar`, `unzip`, moving
files into place, or executing downloaded binaries.

For release assets, I like both layers:

- `SHA256SUMS` as the simple manual verification path;
- GitHub Release asset digests as an extra source of truth for installers and
  native CLIs that already parse release metadata.

When both exist, they should agree. If they do not agree, the installer should
fail closed.

GitHub release verification docs:
https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/verify-release-integrity

## What Verified Commits Communicate

The Verified badge on GitHub is useful, but it is easy to overstate.

It means GitHub verified the signature on the commit or tag against a configured
key. It helps communicate that a source change or release tag came from the
expected person or automation identity.

It does not mean:

- the code is bug-free;
- the release artifact was built correctly;
- the machine that built it was clean;
- the installer is safe to run without reading it.

That is why I treat commit and tag signing as one layer, not the whole system.

## Why 1Password Is In The Setup

I wanted signing keys backed up without making every automated build wait for a
human approval prompt.

For interactive human signing, the 1Password SSH agent is nice. It keeps keys in
1Password and can reduce repeated prompts.

For unattended agents or release automation on a secure machine, I prefer a
file-backed SSH signing key with strict permissions, then I store a recovery copy
in 1Password. In that mode, 1Password is the backup, not the live signing path.

That distinction matters. If a release job needs me to unlock 1Password every
time, the release process is not actually unattended.

## The Reusable Prompt

I turned the pattern into a skill so I do not have to re-think this every time a
project needs a proper installer.

The prompt I would use on another repo is here:

https://github.com/danielraffel/verified-installer-skill/blob/main/prompts/apply-to-existing-repo.md

The skill asks the agent to inspect the existing release flow first. That is
important because different projects hide downloads in different places:

- shell installers;
- PowerShell installers;
- upgrade commands;
- SDK downloads;
- plugin installers;
- cache warmers;
- release workflows.

If the installer verifies the first binary but the CLI later downloads an
unverified SDK, the setup is incomplete.

## Why This Is Worth Doing

This does not make `curl | sh` magically safe. It gives people better options.

A new user can still paste the one-line install command.

A cautious user can download the installer, verify its checksum, read it, and
run it.

A release maintainer can prove that release assets have stable hashes, that
GitHub sees the release as immutable, that artifacts have provenance, and that
source refs are signed by the expected identity.

That is the balance I wanted: simple by default, verifiable when it matters,
repeatable enough that I can apply it to the next project without starting from
scratch.
