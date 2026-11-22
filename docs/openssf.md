# OpenSSF alignment

Gjallar uses OpenSSF projects and standards for **its own** supply chain, not
only for the images it scans. Status as of 2026-11-22.

## OpenSSF projects in use

| Project | How Gjallar uses it |
|---|---|
| **Scorecard** | `scorecard.yml` in the core repo; runs once the repo is public (Scorecard cannot read private repos with the default token) |
| **Sigstore** (cosign, Fulcio, Rekor) | Release images, SBOMs and the Helm chart are signed keyless when the repo is public; Gjallar itself signs your SBOMs as cosign attestations (`workers.sbom.attest`) |
| **SLSA** | Build provenance for every release image and CLI binary (`actions/attest-build-provenance`, BuildKit `provenance: mode=max`) |
| **OpenVEX** | Accepted risks are served as OpenVEX (`/api/v1/images/{digest}/vex`) |
| **OSV** | Offline OSV-Scanner as a second vulnerability source for scanned images (`workers.vuln.osv`); `govulncheck` (OSV-backed) gates Gjallar's own CI |
| **GUAC** | Reads the SBOM and VEX attestations Gjallar signs next to your images (its OCI collector) |
| **Security Baseline (OSPS)** | Self-assessment below |
| **Concise Guide / CVD guide** | `SECURITY.md` follows the OpenSSF coordinated vulnerability disclosure guide |

## OSPS Baseline self-assessment

| Area | Control (summary) | Status |
|---|---|---|
| Access control | MFA for everyone with write access | **You:** require 2FA in the `heimdallr-dev` org settings |
| Access control | Protect the primary branch (no force-push, no deletion, review before merge) | **You:** add a branch ruleset on `main` in each repo |
| Build & release | CI does not run untrusted code with secrets; workflows least-privilege | Done: `permissions` set per job, `persist-credentials: false` |
| Build & release | Dependencies and actions pinned | Done: Actions pinned by SHA, engines and tools by version + checksum, Go modules vendored and verified |
| Build & release | Releases are signed and carry provenance and an SBOM | Done (public Sigstore once public; GitHub attestations while private) |
| Documentation | User guide, contribution guide, security policy | Done: gjallar-docs, CONTRIBUTING.md, SECURITY.md |
| Governance | Roles and decision process documented | Partial: CONTRIBUTING describes review; add a GOVERNANCE.md when there is more than one maintainer |
| Legal | OSI licence on every repo; DCO for contributions | Done: Apache-2.0 everywhere; DCO sign-off on every commit |
| Quality | Automated tests in CI; dependency list available | Done: unit, e2e, chart lint, OPA and gator tests; SBOM per release |
| Security assessment | Threat model / security model documented | Done: architecture.md "Security model"; network egress table |
| Vulnerability management | Private reporting; advisories published | **You:** enable *private vulnerability reporting* in each repo's security settings (SECURITY.md points to it) |
| Vulnerability management | Known vulnerabilities in dependencies fixed before release | Done: `govulncheck` in CI found 55 and they were fixed (Go 1.26.11 and patched modules) |

## Next OpenSSF steps

- **Best Practices badge** (bestpractices.dev): file the self-assessment; most
  "passing" criteria are met by the table above.
- **Allstar** (OpenSSF GitHub app): install on the org to keep branch
  protection, SECURITY.md and binary-artifact rules enforced automatically.
- **GUAC / protobom**: export SBOMs and findings to GUAC for organisation-wide
  supply-chain graphs (CycloneDX/SPDX already available per image).
