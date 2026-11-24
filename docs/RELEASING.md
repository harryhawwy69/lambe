# Releasing lambe

The git tag `vX.Y.Z` ([SemVer 2.0](https://semver.org/spec/v2.0.0.html)) is the
single source of truth for a version. Everything else is derived from it:

| Artifact | Derived from the tag by |
| --- | --- |
| `lambe` binaries, archives, `checksums.txt` | goreleaser (`.goreleaser.yaml` ldflags `main.version/commit/date`) |
| `lambe version` / `--json` | the same ldflags; `go install ...@vX.Y.Z` reads the tag from build info |
| `LambeDash.app` (`CFBundleShortVersionString`, `CFBundleVersion`) | `scripts/package-dashboard.sh` (tag at `HEAD`, or `X.Y.Z` argument) |
| GitHub release notes | the matching `CHANGELOG.md` section |

Development builds (a checkout, `go build`) report `vX.Y.Z-dev+<shortsha>`
(`.dirty` appended for uncommitted changes). `X.Y.Z` is the release the build
is heading toward, so it sorts below that release. `lambe version --json`
prints `version`, `commit`, `date`, and `api_version`.

## SemVer policy while on 0.x

Before 1.0.0 anything may change, but we keep the discipline:

- **Minor (0.Y.0)**: a new feature or any breaking change to the public API
  below, or a removal that finished its deprecation window.
- **Patch (0.9.Z)**: bug fixes, performance, docs, dashboard polish, and
  additive changes that do not alter existing behavior.
- **1.0.0** is cut when the public API below is declared stable; after that,
  breaking changes need a major bump.

Pre-releases use `vX.Y.Z-rc.N`; goreleaser publishes them as normal releases,
so mark them as pre-release on GitHub.

## Public API

Breaking one of these needs a minor bump (major after 1.0) and a CHANGELOG
entry under "Changed" or "Removed":

1. **CLI**: subcommands, flags, exit codes, and `lambe version --json` keys.
2. **`lambe.yaml` schema**: keys, defaults, and precedence. Adding an optional
   key is not breaking.
3. **State API contract**: routes and DTOs in `docs/api/state-api.md`, pinned
   by `testdata/contract/v1/`. It is versioned **separately** from the binary
   as `api_version` (the integer `httpstate.SchemaVersion`, also carried as
   `schema_version` in every response). Additive fields keep the number; a
   breaking wire change bumps it, ships under a new `/api/vN/` prefix, and
   `/api/v(N-1)/` keeps working for at least one minor release. Clients should
   check `api_version` from `lambe version --json` before attaching.

Not public: Go packages under `internal/`, log text, TUI layout, dashboard
visuals, and the `WORKFLOW.md` Symphony semantics beyond what the spec fixes.

## Deprecation policy

Deprecate in one minor release (CHANGELOG "Deprecated", a runtime warning
where practical), keep working for at least one further minor release, then
remove in a later minor release and list it under "Removed".

## Release steps

1. **Changelog.** On a PR to `master`, move the `[Unreleased]` entries into a
   new `## [X.Y.Z] - YYYY-MM-DD` section (Added / Changed / Deprecated /
   Removed / Fixed, plain language), leave a fresh `[Unreleased]`, and update
   the compare links at the bottom of `CHANGELOG.md`. Merge once CI is green.
2. **Validate and tag.** From a clean `master` equal to `origin/master`:
   ```sh
   scripts/release.sh X.Y.Z          # dry run: checks tree, branch, changelog, CI
   scripts/release.sh X.Y.Z --tag    # creates the annotated tag locally
   git push origin vX.Y.Z
   ```
   `scripts/release.sh` is a convenience; the checks that matter also run in
   CI. A `v*` tag whose CHANGELOG section is missing fails `ci` and `release`.
3. **CLI release.** Pushing the tag runs `.github/workflows/release.yml`:
   goreleaser builds linux/darwin x amd64/arm64 archives, `.deb`/`.rpm`/`.apk`
   packages (nfpms), an SBOM per archive (syft), `checksums.txt` with a keyless
   cosign bundle per artifact (`<file>.sigstore.json`, including `checksums.txt`), pushes the Homebrew formula
   to `andrew-bierman/homebrew-tap`, and publishes the GitHub release using the
   CHANGELOG section as notes. The workflow then attests build provenance for
   everything in `checksums.txt`.

   Requirements: the `HOMEBREW_TAP_DEPLOY_KEY` Actions secret (private half of a
   write deploy key on `andrew-bierman/homebrew-tap`; rotate by generating a new
   ed25519 pair, replacing the deploy key and the secret). Third-party actions
   are pinned to commit SHAs with the tag in a comment.
4. **Mac app.** On a Mac with Xcode, from the tagged commit:
   ```sh
   git checkout vX.Y.Z
   scripts/package-dashboard.sh              # reads the version from the tag
   ```
   The script stamps `CFBundleShortVersionString` and `CFBundleVersion` with
   `X.Y.Z`, checks the app icon, builds a universal (arm64 + x86_64) app,
   signs it, verifies the signature, and writes `LambeDash.zip` and
   `LambeDash.zip.sha256` under `apps/dashboard/.build/package/`.

   - **Without a Developer ID** (the default today) the app is signed ad hoc
     with the hardened runtime and is not notarized. Gatekeeper blocks a
     downloaded copy until the user right-clicks it and chooses Open, or
     removes the quarantine flag; the release notes must say so (copy the
     "Signing, and opening it the first time" section of
     `apps/dashboard/README.md`). Never sign a release with a personal or
     self-signed local identity.
   - **With a Developer ID** set `APPLE_SIGNING_IDENTITY` to the
     "Developer ID Application: ..." identity, plus either
     `APPLE_NOTARY_PROFILE` (a `notarytool store-credentials` profile) or
     `APPLE_ID`, `APPLE_TEAM_ID` and `APPLE_APP_SPECIFIC_PASSWORD`. The same
     command then signs, submits to Apple, waits, staples the ticket, and
     re-packs; the result opens with no Gatekeeper prompt.

   Then attach both files to the same release:
   ```sh
   gh release upload vX.Y.Z apps/dashboard/.build/package/LambeDash.zip \
     apps/dashboard/.build/package/LambeDash.zip.sha256
   ```
   This stays manual because signing and notarization need credentials that
   are not in CI.

   Note for release notes: the app's run controls are navigation only (Rex,
   Terminal, copy, Finder) and read-only by design until the state API gets
   mutating actions.
5. **Verify.**
   ```sh
   gh release download vX.Y.Z --dir /tmp/lambe-verify   # use a scratch dir
   cd /tmp/lambe-verify && shasum -a 256 -c checksums.txt --ignore-missing
   shasum -a 256 -c LambeDash.zip.sha256
   ```
   See [Verifying a release](#verifying-a-release) for signatures and attestations.
6. **Install check.** Unpack the archive for your platform and confirm
   `./lambe version --json` reports `"version": "vX.Y.Z"` and the expected
   `api_version`; open `LambeDash.app` (right-click Open for an ad-hoc build) and confirm About
   shows `X.Y.Z`.

## Verifying a release

Every artifact (archives, `.deb`/`.rpm`/`.apk`, SBOMs, `checksums.txt`) has a
keyless cosign bundle named `<file>.sigstore.json` next to it (signed from
v0.12.0; v0.10.1 to v0.11.x signed only `checksums.txt`). Verify a single file:

```sh
cosign verify-blob lambe_X.Y.Z_linux_amd64.deb \
  --bundle lambe_X.Y.Z_linux_amd64.deb.sigstore.json \
  --certificate-identity-regexp '^https://github.com/andrew-bierman/lambe/\.github/workflows/release\.yml@refs/tags/v' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
gh attestation verify lambe_X.Y.Z_linux_amd64.deb --repo andrew-bierman/lambe
```

or verify `checksums.txt` once and check the rest against it:

```sh
cosign verify-blob checksums.txt --bundle checksums.txt.sigstore.json \
  --certificate-identity-regexp '^https://github.com/andrew-bierman/lambe/\.github/workflows/release\.yml@refs/tags/v' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
shasum -a 256 -c checksums.txt --ignore-missing
```

Each archive has an SBOM (`*.sbom.json`) alongside it. The `release.yml` manual
dry run skips signing (`--skip=sign`) and publishes nothing; `signs:` uses
`artifacts: all`, so new artifact types are signed automatically.

## Reproducible builds

Release binaries, archives and packages are byte-for-byte reproducible from a
commit: `-trimpath`, `-buildid=`, `main.date` from the commit date
(`{{ .CommitDate }}`, not build time), and `mod_timestamp`/archive/nfpm mtimes
pinned to the commit. To verify, build twice (the second time from a fresh clone
in another directory) and compare hashes. SBOMs and signatures are skipped
because they are not part of the comparison:

```sh
goreleaser release --snapshot --clean --skip=publish,sign,sbom
(cd dist && shasum -a 256 checksums.txt lambe_*.tar.gz lambe_*.deb lambe_*.rpm lambe_*.apk) > /tmp/a.txt
# repeat in a second clone of the same commit, then:
diff /tmp/a.txt /tmp/b.txt && echo reproducible
```

Use the same Go toolchain (`mise.toml`) for both runs. A published release can be
checked the same way: build the tag's commit and compare against
`checksums.txt`.

## Homebrew

`brew install andrew-bierman/tap/lambe` installs the formula that goreleaser
pushes to `andrew-bierman/homebrew-tap` on every release (it depends on the
`beads` formula for `bd`). goreleaser marks `brews` deprecated in favor of
`homebrew_casks`, but a cask would drop Linux support and, for an unsigned
binary, need a quarantine-stripping hook; the formula stays until that breaks.

**No LambeDash cask.** `LambeDash.app` is ad-hoc signed and not notarized.
Homebrew rejects casks that fail Gatekeeper, so a cask would be broken or need
a quarantine workaround. Revisit once a Developer ID and notarization are in
place (`scripts/package-dashboard.sh` already supports them); until then the
Mac app stays a manual release asset.

## Legacy

Before this policy the Mac app shipped under its own `dashboard-v1.0` tag and
release. That tag and release are kept for history and are no longer used: the
app now ships as an extra asset on the matching `vX.Y.Z` release.
