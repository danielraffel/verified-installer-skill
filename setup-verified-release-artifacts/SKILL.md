---
name: setup-verified-release-artifacts
description: >-
  Set up verification for GitHub Release artifacts without adding a curl
  installer. Use when a repo publishes downloadable packages, examples,
  plugins, apps, archives, installers, or binaries such as .pkg, .dmg, .zip,
  .tar.gz, .deb, .rpm, .AppImage, .exe, .msi, .apk, .aab, or future iOS/mobile
  artifacts, and Codex should add SHA256SUMS, GitHub artifact attestations,
  immutable-release verification, release asset digest checks, signed source
  refs, and concise README verification instructions. Use this only when there
  is no install.sh/install.ps1 or installer path to create or harden; if the
  repo has both an installer and downloadable release artifacts, use
  setup-verified-installer instead.
---

# Setup Verified Release Artifacts

## Overview

Use this skill for repos where users download files from GitHub Releases, but
there is no `install.sh` or shell installer to harden. The goal is to make
release assets easy to verify and to describe the trust layers honestly.

If a repo has both an installer and direct release downloads, use
`setup-verified-installer`. That skill covers the installer path and the
release artifacts it downloads or publishes. This skill is the narrower
release-only path.

Keep these layers separate:

- platform signing/notarization says whether the OS trusts the package;
- `SHA256SUMS` and GitHub release asset digests say the downloaded bytes match
  what was published;
- artifact attestations say which GitHub Actions workflow produced or handled
  those bytes;
- immutable releases lock published tags and assets after publication;
- signed commits/tags identify the source ref as coming from a configured
  maintainer or bot identity.

If the repo needs a `curl | sh` installer, use `setup-verified-installer`
instead. If the repo only publishes release packages such as `.pkg`, `.dmg`,
`.zip`, `.tar.gz`, `.exe`, `.msi`, `.deb`, `.rpm`, `.AppImage`, `.apk`, or
`.aab`, use this skill.

## Workflow

1. Inspect the release surface.
   - Read `README`, `AGENTS.md`, `CLAUDE.md`, package scripts, release scripts,
     `.github/workflows/*`, and any docs that mention downloads.
   - Query the latest release with `gh release view` or `ghapp release view`.
   - List release assets and capture names, sizes, GitHub `digest` fields,
     download URLs, and whether a `SHA256SUMS` asset already exists.
   - Check whether releases are created by Actions, by Shipyard, by a local
     command, or manually in the GitHub UI.
   - Check whether release tags are signed, branch/tag protections exist, and
     immutable releases are enabled.

2. Classify the release pipeline before editing.
   - **Actions-built:** the workflow builds/signs/packages the final release
     artifacts. This is the best path for provenance. Add checksums,
     `actions/attest`, draft-first release publishing, and README verification.
   - **Hybrid:** a workflow receives or assembles already-built final bytes, then
     uploads them. Attestation can honestly prove the workflow handled those
     bytes, not that it built them from source. Say that in docs.
   - **Manual/local-built:** artifacts are built and uploaded outside Actions.
     Add `SHA256SUMS`, release digests, platform verification, and optional
     immutable releases. Do not claim build provenance until the final artifact
     path runs through Actions.
   - **Source-only:** no downloadable binary/package assets. Do not add
     heavyweight release verification unless the repo is about to publish assets.

3. Decide the minimum good target for this repo.
   - Always prefer `SHA256SUMS` for plain `curl` plus `shasum`/`sha256sum`.
   - Use GitHub release asset `digest` fields as an additional verification
     source, not the only user-facing manual path.
   - Add artifact attestations only for final release bytes available in the
     workflow workspace.
   - Enable immutable releases only when the workflow can create a draft, attach
     all assets including `SHA256SUMS`, then publish.
   - Keep source-ref signing as separate hardening; do not block checksum work
     on it unless the user asks for the full provenance stack.

## Release Asset Targets

Cover all user-facing files that someone might download and run, install, load,
or redistribute:

- macOS: `.pkg`, `.dmg`, `.zip`, plugin bundles wrapped in an archive;
- Windows: `.exe`, `.msi`, `.zip`;
- Linux: `.tar.gz`, `.tgz`, `.zip`, `.deb`, `.rpm`, `.AppImage`;
- Android: `.apk`, `.aab`;
- iOS or Apple-platform app distribution: `.ipa`, `.xcarchive`, signed/notarized
  archives when they are published as release assets.

Do not attest or checksum generated logs, screenshots, temporary build
artifacts, or source files unless they are published as release assets.

## Checksum Pattern

Generate checksums after all release assets exist and before publication:

```bash
mkdir -p release-assets
cp dist/* release-assets/
(
  cd release-assets
  rm -f SHA256SUMS checksums.sha256
  shasum -a 256 * | LC_ALL=C sort > SHA256SUMS
  shasum -a 256 -c SHA256SUMS
)
```

For Linux-centric docs, also show GNU verification:

```bash
sha256sum --check SHA256SUMS
```

If the project already uses another checksum filename, either migrate to
`SHA256SUMS` or upload the old name as an alias. Do not make users guess.

## Attestation Pattern

Use artifact attestations when the final release bytes are in a GitHub Actions
workspace. Add the narrowest needed permissions to the job that has the final
files:

```yaml
permissions:
  contents: read
  id-token: write
  attestations: write
```

Then attest final artifacts after packaging/signing/notarization and after any
last byte-changing step:

```yaml
- name: Attest release artifacts
  uses: actions/attest@v4
  with:
    subject-path: |
      dist/*.pkg
      dist/*.dmg
      dist/*.zip
      dist/*.tar.gz
      dist/*.deb
      dist/*.rpm
      dist/*.AppImage
      dist/*.exe
      dist/*.msi
      dist/*.apk
      dist/*.aab
```

Attesting before signing, notarizing, compressing, or repackaging is wrong: the
attestation would cover bytes users do not download.

For manual/local-built releases, do not invent provenance. Either:

- leave attestations out and document checksum/platform verification; or
- move the final package assembly/signing/upload path into Actions; or
- document a hybrid claim such as "this workflow attested the uploaded bytes,"
  not "this workflow built the package from source."

## Draft-First Release Pattern

For immutable releases, publish in this order:

1. create a draft release for the tag;
2. upload every user-facing asset;
3. generate and upload `SHA256SUMS`;
4. run local checksum verification;
5. publish the draft.

Publishing an immutable release too early prevents adding missing assets later.
Do not enable immutable releases until the release workflow is draft-first.

Check or enable immutable releases with:

```bash
gh api repos/OWNER/REPO/immutable-releases
gh api -X PUT repos/OWNER/REPO/immutable-releases --silent
```

Use `ghapp` in Daniel's Shipyard environment when available.

## README Pattern

Keep verification concise and near the download instructions. For a macOS `.pkg`
release:

````markdown
### Verify the download

For an additional check before installing:

```bash
version=0.2.5
asset="PROJECT-${version}.pkg"
base="https://github.com/OWNER/REPO/releases/download/v${version}"

curl -fSLO "$base/$asset"
curl -fSLO "$base/SHA256SUMS"
awk -v file="$asset" '$2 == file { print }' SHA256SUMS | shasum -a 256 -c -
```

If the release publishes artifact attestations, you can also verify provenance:

```bash
gh attestation verify "$asset" \
  -R OWNER/REPO \
  --signer-workflow github.com/OWNER/REPO/.github/workflows/release.yml
```

The checksum confirms the downloaded file matches the published release asset.
The attestation links the file to the GitHub Actions workflow that produced or
handled it. macOS will still perform its normal package signature and
notarization checks when you open the installer.
````

For longer explanations, put details in a collapsed `<details>` block. Do not
let verification prose bury the primary download/build instructions.

## Platform Checks

Add platform-specific commands only when they match the asset type:

```bash
# macOS package signature / Gatekeeper
pkgutil --check-signature Project.pkg
spctl --assess --type install -vv Project.pkg

# macOS app or dmg, when applicable
codesign --verify --deep --strict --verbose=2 Project.app
spctl --assess --type open --context context:primary-signature -vv Project.dmg

# Windows
powershell -Command "Get-FileHash .\\Project.msi -Algorithm SHA256"
powershell -Command "Get-AuthenticodeSignature .\\Project.msi"

# Linux
sha256sum --check SHA256SUMS
dpkg-deb --info Project.deb
rpm -K Project.rpm

# Android
apksigner verify --verbose Project.apk
```

Treat platform signing as complementary. It does not replace `SHA256SUMS` or
artifact attestations.

## Validation Checklist

Before calling the setup done:

- `gh release view TAG --json assets` shows every user-facing asset and
  `SHA256SUMS`.
- Every release asset has a GitHub `digest` value, unless it predates GitHub's
  release digest feature.
- `gh release download TAG --dir verify-assets` downloads the intended assets.
- `shasum -a 256 -c SHA256SUMS` or `sha256sum --check SHA256SUMS` passes.
- `gh attestation verify ASSET -R OWNER/REPO --signer-workflow ...` passes for
  each attested asset class.
- `gh release verify TAG -R OWNER/REPO` passes when immutable releases are
  enabled for releases created after immutability was enabled.
- The README verification commands work in a fresh temporary directory.
- The docs do not claim build provenance for assets built entirely outside
  GitHub Actions.
- Signed commit/tag checks are validated separately if the task includes source
  provenance.

## Example Triage

For a repo like `danielraffel/pulp-classic-effects` that publishes a notarized
`.pkg` manually:

- current baseline: GitHub release asset digest exists;
- useful immediate upgrade: upload `SHA256SUMS` and add README verification;
- next provenance step: move `.pkg` production or final package handling into
  Actions, then add `actions/attest`;
- optional hardening: enable immutable releases after switching to draft-first
  publishing;
- do not claim the `.pkg` was built by Actions until the final `.pkg` bytes are
  actually created or handled by an Actions workflow.
