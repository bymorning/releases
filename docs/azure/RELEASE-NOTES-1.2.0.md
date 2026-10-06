# ByMorning 1.2.0 release notes

ByMorning 1.2.0 for Microsoft Azure runs on Azure Kubernetes Service in **Azure Government** and in **global
Azure**. This package installs it with Terraform and Kubernetes manifests; see [README.md](README.md).

| Image          | Tarball                                             | Digest                 |
| -------------- | --------------------------------------------------- | ---------------------- |
| Gateway        | `release/bymorning-a8470c80cd9b.tar`                  | `sha256:02710e58b12563e46eb60b8d4ecf298cf65648574a006d5a07c42d4251ef45e3`   |
| Code execution | `release/bymorning-code-interpreter-a8470c80cd9b.tar` | `sha256:30bf212923537164a83736c2dd1870c5b92744fef93c37ed1371b9f4b113a826` |

Both images are built from source commit `a8470c80cd9b` and signed with the ByMorning release key (`release/cosign.pub`).
SBOMs, vulnerability scans, the SLSA provenance statement, the OpenVEX statement, and the security triage are in
`release/`; see the README's [Release evidence](README.md#release-evidence) section.

## What's new in 1.2.0

- **Project archiving.** Archive a Project from its settings; restore it from **Create project**. An archived Project
  is read-only: no new Sessions or messages, and its Scheduled Tasks don't run. The Default Project can't be archived.
- **Project files.** Upload, download, rename or move, and delete files and folders from the file tree in a Session's
  right sidebar. Moves are recorded in the audit log as `file.move`.
- **Archived Sessions.** Archiving asks for confirmation and offers Undo; a Project's page lists its archived Sessions
  with **Restore**.
- **Integrations.** The Console's former MCP page is now **Integrations**: connected services and MCP servers in one
  list, with one picker to add more.
- **Administration.** Inviting your own address can no longer demote a Workspace's only admin; errors in SSO,
  Grant admin, and invitation dialogs are shown; Console pages are consistent and usable on narrow screens; Routing
  and Installation Policy warn before discarding unsaved changes; budgets show when spending is over the limit.
- **Reliability.** Faster Session loading while work is running, live file tree and preview updates, plain-language
  tool failures, and startup on Azure stops with a clear error after 60 seconds if Key Vault or Blob Storage stalls.

## Upgrading from 1.1.0

Follow the README's [Upgrade](README.md#upgrade) section. In short: verify and mirror the images with
`release/mirror.sh`, take a database backup, re-run `overlay.sh` from **this** package's `installers/azure` with the new
digest, and `kubectl apply -k` once. Apply the 1.2.0 manifests together with the 1.2.0 image: the image requires the
`STORAGE_PROVIDER` and `CIPHER_BACKEND` settings that the 1.2.0 manifests add, and names any missing one in its startup
log. One additive database migration runs automatically. To roll back, restore the backup taken before the upgrade and
re-apply the previous digest.

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
