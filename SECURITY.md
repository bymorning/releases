# Security

## Reporting a vulnerability

Report suspected vulnerabilities in ByMorning privately through <https://bymorning.ai>. Do not open a public issue.
Include the release version, the image digest if known, and steps to reproduce. We acknowledge reports and keep
you informed until the issue is resolved.

## How releases are built

- **Source.** Each release is built from a single tagged revision of the ByMorning source. The revision is recorded
  in `release/image-*.txt` inside the package and in the OCI image labels.
- **Gateway image.** Red Hat Universal Base Image 9 (minimal) with Node.js 22 linked against the system OpenSSL,
  running with the RHEL OpenSSL FIPS provider enabled. The image runs as an unprivileged user, has `npm` and build
  tooling removed, and listens on one port.
- **Code execution image.** A separate, minimal image used only for sandboxed code execution. It contains no
  ByMorning code and runs as an unprivileged user inside a pod with no network access. It carries no FIPS claim.
- **Evidence.** For every image we publish CycloneDX and SPDX SBOMs, a Grype vulnerability scan, and a written
  triage of every Critical and High finding. The tool versions used are listed in `tools.txt` beside the reports.
- **Signing.** Image tarballs and every `SHA256SUMS` are signed with `cosign` using an ECDSA P-256 key held in a
  hardware-backed key management service. The public key is [`cosign.pub`](cosign.pub) in this repository and is
  attached to every release. Any change of key will be announced in the release notes and reflected here.

Source-level evidence (source SBOM, source dependency scan, static analysis, secret scan), SLSA provenance, and the
signed OpenVEX statement for each release are available to licensed customers on request.

## Vulnerability handling

Findings are triaged against the shipped image rather than the scanner output alone. A finding is recorded as
_not affected_ only with a stated justification (for example, the component is removed from the runtime image or
the vulnerable code is not reachable). Findings we accept are recorded with the exposure, the mitigation, and the
release that will fix them. Operating-system findings with no vendor fix at build time are picked up with base-image
updates in the next release.

## Supported versions

Security fixes are delivered in the current release line. Upgrade guidance is in each package's installation guide.
