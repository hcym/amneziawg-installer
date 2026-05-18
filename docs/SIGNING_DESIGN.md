# Release signing design (v5.14+ proposal)

Status: **DRAFT** - awaiting user-action (offline keypair generation) before activation.

## Why

Modern open-source security practice (NixOS, signify-based tools, libsodium ecosystem) ships releases with detached cryptographic signatures so users can verify a downloaded script has not been tampered with on the path from GitHub to their server. Right now `install_amneziawg.sh` is fetched via HTTPS from `raw.githubusercontent.com`, which means trust is rooted in GitHub's TLS chain + GitHub's account security alone. Adding a maintainer-controlled signature gives an independent verification path:

- TLS only proves "the bytes came from GitHub". A signature proves "the bytes were signed by the holder of the private key", which lives offline on the maintainer's machine and is never exposed to GitHub Actions.
- If the GitHub account is compromised and an attacker uploads a malicious script, the absent or invalid signature fails verification on the user's side. This is the gap GitHub TLS alone does not cover.

Competitor `pwnnex/ByeByeVPN` (303 stars, viral growth +48/week) already ships `minisign` signatures and an SBOM with each release. Cost: ~2-4 hours one-time setup + ~30 sec per release.

## Tool choice: minisign

| Option | Pros | Cons |
|---|---|---|
| `minisign` ([jedisct1/minisign](https://github.com/jedisct1/minisign)) | Curve25519, single 32-byte key pair, no GPG keyring drama, Ed25519 signing, `minisign -V` is one command for users | Less ubiquitous than GPG, must be installed (`apt install minisign`) |
| `cosign` keyless via Sigstore | No key management, federated trust through Fulcio | Newer dependency, requires Sigstore infra availability at verification time, harder to verify offline |
| GPG signed tags + commits | Already supported by GitHub UI ("Verified" badge) | Does not cover downloaded raw script files, only commit/tag objects |

**Decision:** minisign. Smallest moving parts, offline-friendly verify, no third-party trust roots, well-understood by security-conscious sysadmins.

## Threat model

Covered:
- Tampering with `install_amneziawg.sh` or `install_amneziawg_en.sh` between GitHub Releases and the user's `wget` call.
- Compromise of the GitHub account leading to a malicious replacement upload (the attacker cannot forge a signature without the offline secret key).

NOT covered:
- Compromise of the maintainer's offline machine where the private key is stored.
- Social engineering tricking the user into running a different command.
- Supply chain attacks on the AmneziaWG kernel module or the Amnezia PPA (out of scope - those are upstream concerns).

## Keypair generation (one-time, USER-ACTION)

The private key MUST be generated offline by the maintainer and MUST NEVER leave that machine.

```bash
# On a clean, network-isolated machine if possible:
minisign -G -p amneziawg-installer.pub -s amneziawg-installer.key

# Choose a strong password. Write it down somewhere physical.
# Backup the .key file to encrypted offline storage (e.g., encrypted USB stick).
```

Generated files:
- `amneziawg-installer.pub` (public key) - 56-byte file, safe to commit to the repository as `KEYS.txt` or `KEYS/amneziawg-installer.pub`.
- `amneziawg-installer.key` (private key) - encrypted with the password. NEVER commit. NEVER upload to GitHub Secrets (defeats the purpose - signing must be local to the maintainer's machine).

## Signing flow

Per release, after `git tag vX.Y.Z` but before `git push origin vX.Y.Z`:

```bash
# Sign both installers and both management helpers:
minisign -Sm install_amneziawg.sh -s ~/.minisign/amneziawg-installer.key
minisign -Sm install_amneziawg_en.sh -s ~/.minisign/amneziawg-installer.key
minisign -Sm manage_amneziawg.sh -s ~/.minisign/amneziawg-installer.key
minisign -Sm manage_amneziawg_en.sh -s ~/.minisign/amneziawg-installer.key
minisign -Sm awg_common.sh -s ~/.minisign/amneziawg-installer.key
minisign -Sm awg_common_en.sh -s ~/.minisign/amneziawg-installer.key
```

Produces `*.minisig` files alongside each script.

Then attach them to the GitHub Release as assets (manually via `gh release upload`, or via the workflow described below).

## Workflow integration (proposal)

Two options, pick one when activating:

### Option A: Manual asset upload (lighter)

After `git push origin vX.Y.Z`, the existing `release.yml` creates the release from `CHANGELOG.en.md`. Add a manual step:

```bash
gh release upload vX.Y.Z \
  install_amneziawg.sh install_amneziawg.sh.minisig \
  install_amneziawg_en.sh install_amneziawg_en.sh.minisig \
  manage_amneziawg.sh manage_amneziawg.sh.minisig \
  manage_amneziawg_en.sh manage_amneziawg_en.sh.minisig \
  awg_common.sh awg_common.sh.minisig \
  awg_common_en.sh awg_common_en.sh.minisig
```

Pros: zero CI changes, signatures generated on the trusted maintainer machine. Cons: extra manual step per release.

### Option B: CI uploads signatures generated locally (asymmetric)

Maintainer generates `*.minisig` files locally, commits them transiently to a `signing/` directory (gitignored elsewhere), tags. A new job in `release.yml` reads them and uploads as assets. Same trust model as Option A - just automates the upload step.

The signing of the files NEVER happens in GitHub Actions. The private key is never exposed to Actions. This is intentional and the whole point.

A draft of Option B is committed at `docs/release-sign.yml.draft` for review. It is NOT placed in `.github/workflows/` until the public key is published.

## User-side verification

Document in README "Verifying releases" section:

```bash
# 1. Install minisign:
sudo apt install minisign           # Ubuntu/Debian
# or:
brew install minisign                # macOS

# 2. Fetch the public key from the repository (one time):
curl -O https://raw.githubusercontent.com/bivlked/amneziawg-installer/main/KEYS.txt

# 3. Fetch the installer + signature:
TAG=v5.14.0
curl -LO "https://github.com/bivlked/amneziawg-installer/releases/download/$TAG/install_amneziawg_en.sh"
curl -LO "https://github.com/bivlked/amneziawg-installer/releases/download/$TAG/install_amneziawg_en.sh.minisig"

# 4. Verify:
minisign -V -p KEYS.txt -m install_amneziawg_en.sh -x install_amneziawg_en.sh.minisig
# Expected: "Signature and comment signature verified"

# 5. If verified - now you can install:
sudo bash ./install_amneziawg_en.sh
```

## Implementation checkpoints

Activation steps, in order:

1. **USER**: Generate offline keypair with `minisign -G` on a trusted machine. Backup the private key to encrypted offline storage. Set a strong password.
2. **USER**: Hand over the public key file (`*.pub`) for commit to the repository as `KEYS.txt`.
3. Add `docs/SIGNING_DESIGN.md` (this file). DONE in this commit.
4. Add README section "Verifying releases" with placeholder link to this design doc. DONE.
5. Add the draft workflow `docs/release-sign.yml.draft` for review. DONE.
6. After keypair exists and is published as `KEYS.txt`:
   a. Move `docs/release-sign.yml.draft` to `.github/workflows/release-sign.yml`.
   b. Test on a pre-release tag (e.g., `v5.14.0-rc1`).
   c. Flip the README section from "planned" to "active".
7. Optional follow-up: SBOM generation via `syft` or GitHub's native dependency graph (a separate, smaller task).

## Out of scope (intentionally deferred)

- **SBOM generation**: distinct deliverable. Will be added in a follow-up commit once signing is stable. `syft` is the leading tool; GitHub also auto-generates a dependency graph SBOM which is enough for an initial pass.
- **Signing of ARM prebuilt `.deb` packages** published to the `arm-packages` release: same principle applies but needs a separate flow because the arm-build workflow runs inside Docker via QEMU. Defer.
- **Reproducible builds**: Bash scripts are already self-contained text files - signatures cover them as-is. No build determinism work needed.

## References

- minisign: <https://jedisct1.github.io/minisign/>
- NixOS signify: <https://nixos.org/manual/nix/stable/protocols/signing-prefixes.html>
- pwnnex/ByeByeVPN (prior art in this corner): <https://github.com/pwnnex/ByeByeVPN>
- Discussion of GPG vs minisign for sysadmin scripts: <https://github.com/jedisct1/minisign#why-not-gpg>
