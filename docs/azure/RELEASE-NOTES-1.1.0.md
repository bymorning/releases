# ByMorning 1.1.0 release notes

ByMorning 1.1.0 adds Microsoft Azure support. It runs on Azure Kubernetes Service in
**Azure Government** and in **global Azure**. This package installs it with Terraform and Kubernetes manifests; see
[README.md](README.md).

| Image           | Digest                                                                    |
| --------------- | ------------------------------------------------------------------------- |
| Gateway (Azure) | `sha256:0842ae72ba56aee9ad86b41bddf41124f2c61fd4c8660691b5f211fb2f9b40c9` |
| Code execution  | `sha256:96e4d4d797f85c3ae3715f544370a40fa29dfcf9473406826b3885120d6ee333` |

Both images are signed with the ByMorning release key (`release/cosign.pub`). Image SBOMs, vulnerability scans, and
the security triage are in `release/`; see the README's [Release evidence](README.md#release-evidence) section.

## Licensing

Every installation starts a 30-day trial with every capability enabled. An Installation Admin activates a license on
the **License** page before the trial ends; otherwise the installation becomes read-only until one is activated.
Request a license at <https://bymorning.ai>.

## Sign-in with Microsoft Entra

- A Microsoft Entra tenant in the matching cloud (Government tenant for Azure Government), with a single-tenant
  **Web** app registration.
- Optional ID-token claims **`email`** and **`xms_edov`** must be added under Token configuration.
- Entra sends `xms_edov` (email domain verified) instead of `email_verified`. A user is admitted when the ID token has
  an `email` in an admitted domain and `xms_edov: true`, so verify each admitted domain in the tenant.
- Guest (`#EXT#`) users and users without a `mail` attribute are refused.

## Code execution sandbox

Code runs in short-lived, hardened pods in a dedicated namespace with a deny-all network policy. The cluster's
network policy engine must enforce `NetworkPolicy`; the Gateway refuses to execute code otherwise. Optional Kata
VM isolation is available through the `sandbox_node_pool` setting. See [sandbox/README.md](sandbox.md).

## Amazon Bedrock

Bedrock is optional. The Gateway reaches it either with **web identity federation** (the pod's service account token
is exchanged for AWS credentials; no stored AWS secret) or with a **Bedrock API key** held in a Kubernetes Secret.
Both GovCloud (`aws-us-gov`) and commercial (`aws`) partitions are supported; the partition is your choice.

## Known limits

- **No FIPS claim for the execution image.** The Gateway image uses the RHEL 9 OpenSSL FIPS Provider; code that runs
  inside the execution image, and its userland, carries no FIPS or other compliance claim.
- **Confirm in your subscription:** availability of Azure CNI Powered by Cilium and of Kata pod sandboxing in your
  Azure Government region, and quota for the node sizes you choose.
- **One Gateway replica.** Upgrades replace the pod and cause a short outage.
- **Accepted advisory:** a denial-of-service advisory in the `brace-expansion` package is accepted for this release
  and will be fixed in 1.1.1. See `release/gateway/vulnerability-triage.md`.
- The scan reports in `release/` list operating-system package findings with no vendor fix available at build time;
  `release/gateway/vulnerability-triage.md` gives the position on each Critical and High finding.
