<p align="center">
  <img src="assets/wordmark.svg" alt="ByMorning" width="520">
</p>

# ByMorning releases

ByMorning is a self-hosted gateway between AI and your internal systems. The ByMorning chat client, coding agents,
and your own applications connect through it to reach models and tools. Every request is permission-checked and
logged, and administrators configure providers, integrations, permissions, policies, and budgets in one place.

This repository publishes ByMorning releases: signed container images, the installer for each supported platform,
and the security evidence for every build. Product information is at <https://bymorning.ai>.

## Current release

**ByMorning 1.2.1** — [release page](https://github.com/bymorning/releases/releases/tag/v1.2.1) ·
[release notes](docs/azure/RELEASE-NOTES-1.2.1.md)

| Platform                       | Package                                                                                                                 | Installation guide                 |
| ------------------------------ | ----------------------------------------------------------------------------------------------------------------------- | ---------------------------------- |
| Azure Kubernetes Service (AKS) | [`bymorning-azure-1.2.1.zip`](https://github.com/bymorning/releases/releases/download/v1.2.1/bymorning-azure-1.2.1.zip) | [docs/azure](docs/azure/README.md) |

The Azure package supports Azure Government and global Azure. For other environments, contact ByMorning through
<https://bymorning.ai>.

## What a release contains

Each package is a self-contained directory:

| Path               | Contents                                                                                   |
| ------------------ | ------------------------------------------------------------------------------------------ |
| `README.md`        | Installation guide: prerequisites, configuration, deploy, first sign-in, upgrade, rollback |
| `RELEASE-NOTES.md` | What changed, requirements, known limits                                                   |
| `installers/`      | Terraform module and example root, Kubernetes manifests, and helper scripts                |
| `release/`         | OCI image tarballs, their digests, the signing key, and signed evidence for each image     |
| `licenses.md`      | Third-party license report                                                                 |

Images are shipped as OCI tarballs rather than pulled from a public registry, so you push them into your own
registry and pin them by digest. Every image is accompanied by:

- a CycloneDX and an SPDX software bill of materials;
- a vulnerability scan and a written triage of every Critical and High finding;
- a signed `SHA256SUMS` covering the tarball and the reports beside it.

## Verifying a release

Releases are signed with the ByMorning release key. The public key is pinned in this repository as
[`cosign.pub`](cosign.pub) and is also attached to every release; the two must match.

```sh
curl -fsSLO https://github.com/bymorning/releases/releases/download/v1.2.1/bymorning-azure-1.2.1.zip
curl -fsSLO https://github.com/bymorning/releases/releases/download/v1.2.1/bymorning-azure-1.2.1.zip.bundle
curl -fsSLO https://raw.githubusercontent.com/bymorning/releases/main/cosign.pub

cosign verify-blob --key cosign.pub --bundle bymorning-azure-1.2.1.zip.bundle bymorning-azure-1.2.1.zip
```

Inside the package, the installation guide shows how to verify each image tarball and its evidence with the same
key before pushing it to your registry.

```text
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAErtYUpiz3HufOvMF5nQ5eGBt3Wh4n
H4zDmOn8Lpc8daxigMr7H/hiVlzrYcLQ7RQsM0f4cL3caZgH3SW0UUS8hg==
-----END PUBLIC KEY-----
```

## Licensing

Every installation starts a 30-day trial with every capability enabled. Before the trial ends, an Installation Admin
activates a license on the **License** page by pasting the signed license issued by ByMorning; it is verified
offline, without contacting ByMorning. Once the trial has ended without a license, the installation is read-only
until one is activated. Nothing is deleted.

Request a license, or ask a question, at <https://bymorning.ai>.

## Security

See [SECURITY.md](SECURITY.md) for how releases are built and signed and how to report a vulnerability.

## License

The files in this repository are available under the [MIT License](LICENSE). The ByMorning software distributed
through the releases (container images and the software inside them) is licensed separately under the
[ByMorning Software License Agreement](https://bymorning.ai/license).
