# ByMorning 1.3.0 release notes

ByMorning 1.3.0 for Microsoft Azure runs on Azure Kubernetes Service in **Azure Government** and in **global
Azure**. This package installs it with Terraform and Kubernetes manifests; see [README.md](README.md).

| Image          | Tarball                                             | Digest                 |
| -------------- | --------------------------------------------------- | ---------------------- |
| Gateway        | `release/bymorning-92f9b249d2d9.tar`                  | `sha256:a5f8047523f3b71fd376e167b80955570db9a22161e6abde00ffaed1189c7482`   |
| Code execution | `release/bymorning-code-interpreter-92f9b249d2d9.tar` | `sha256:1c664f22abb77c7c8b6349f8a40b2bb5be115c328e513d0292ef724d84c8fa0a` |

Both images are built from source commit `92f9b249d2d9` and signed with the ByMorning release key (`release/cosign.pub`).
SBOMs, vulnerability scans, the SLSA provenance statement, the OpenVEX statement, and the security triage are in
`release/`; see the README's [Release evidence](README.md#release-evidence) section.

## What's new in 1.3.0

- **Per-person credentials for MCP servers.** An MCP server that authenticates with each person's own token, such as
  Jira or Confluence personal tokens sent in headers, can now be added without sharing one person's token. The
  administrator writes the headers with `${NAME}` where each member's value goes (and `${member.email}` for the
  member's email) and chooses **Member values** when adding the server. Each member then saves their own values from the
  command palette: **Connect integration**, under **MCP servers**. Values are stored encrypted per member, sent only to
  that server, and never shown again. Administrators see which members have saved values, never the values. The Gateway
  must be able to reach the MCP server over HTTPS; with `outbound_type = "userDefinedRouting"`, allow it on your
  firewall.
- **Custom Models.** An administrator can add a Model to an active Provider on AI Gateway → Models → **Add model**: a
  display label and the upstream model name. Prices and a context limit are optional. A Bedrock Model still needs its
  ARNs in the Bedrock settings.
- **Setup and sign-in errors say what to fix.** Setup now reports an identity provider it cannot reach (check the
  Gateway's outbound HTTPS access to `login.microsoftonline.us` or `login.microsoftonline.com`) or refuses (issuer
  mismatch, missing endpoints, no PKCE), and a failed sign-in returns to the login page with a reason (`claims`,
  `domain`, `exchange`, and so on) and a hint. The documentation has a new **Deploy → Microsoft Entra** guide
  (`/deploy/entra`).
- **Shared MCP header values are hidden from members.** Only administrators can read them through the API; members see
  `[redacted]`.
- **Security.** The MCP SDK is updated to 2.3.1 for three advisories published 2026-10-20. `release/mirror.sh` now
  verifies each tarball's cosign signature (when `cosign` is on `PATH`) and its OCI structure before pushing.

1.3.0 includes everything in 1.2.1. 1.2.2 changed only the AWS GovCloud model catalog and does not affect Azure.

## Upgrading

There are no new database migrations and no new Terraform variables or settings.

**From 1.2.1 or 1.2.0:** verify and mirror the images with `release/mirror.sh`, take a database backup, re-run
`overlay.sh` from **this** package's `installers/azure` with the new digest, and `kubectl apply -k`. Apply this
package's manifests together with its image.

**From 1.1.0:** follow the README's [Upgrade](README.md#upgrade) section. The image requires the `STORAGE_PROVIDER` and
`CIPHER_BACKEND` settings that these manifests add, and names any missing one in its startup log. One additive
database migration from 1.2.0 runs automatically.

**Rolling back.** 1.2.1 and earlier cannot read the credentials that member values create. Before re-applying an
earlier image, remove each MCP server that uses member values (Console → Integrations → Remove); removing a server
deletes every member's saved values for it. Otherwise a rollback to 1.2.1 needs no backup restore. Don't roll back
after rotating the Key Vault key after upgrading from 1.1.0: 1.1.0 reads data keys with the newest key version only.

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
- The scan reports in `release/` list operating-system package findings with no vendor fix available at build time;
  `release/gateway/vulnerability-triage.md` gives the position on each Critical and High finding, and
  `release/gateway/evidence/openvex.json` records the same dispositions in OpenVEX form.
