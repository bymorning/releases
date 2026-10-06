# ByMorning 1.2.0 on Azure (AKS)

> This is the installation guide shipped as `README.md` inside
> [`bymorning-azure-1.2.0.zip`](https://github.com/bymorning/releases/releases/download/v1.2.0/bymorning-azure-1.2.0.zip).
> Paths refer to the unpacked package. The [release notes](RELEASE-NOTES-1.2.0.md) and
> [sandbox guide](sandbox.md) are alongside.

## Contents of this package

| Path                | Contents                                                                                      |
| ------------------- | --------------------------------------------------------------------------------------------- |
| `README.md`         | This installation guide                                                                       |
| `RELEASE-NOTES.md`  | What 1.2.0 is, sign-in requirements, known limits                                             |
| `installers/azure/` | Terraform module and example root, Kubernetes manifests, scripts, and `smoke.sh`              |
| `sandbox/`          | Code execution sandbox manifests and their [README](sandbox.md)                        |
| `release/`          | Image tarballs, digests, signing key, `mirror.sh`, and signed evidence (SBOMs, scans, triage) |
| `licenses.md`       | Third-party license report for the Gateway image                                              |

| Image          | Tarball                                               | Digest                                                                    |
| -------------- | ----------------------------------------------------- | ------------------------------------------------------------------------- |
| Gateway        | `release/bymorning-a8470c80cd9b.tar`                  | `sha256:02710e58b12563e46eb60b8d4ecf298cf65648574a006d5a07c42d4251ef45e3` |
| Code execution | `release/bymorning-code-interpreter-a8470c80cd9b.tar` | `sha256:30bf212923537164a83736c2dd1870c5b92744fef93c37ed1371b9f4b113a826` |

Both images were built from source commit `a8470c80cd9b`. Evidence: `release/gateway/evidence/` (Gateway) and
`release/code-interpreter/evidence/` (code execution), signed with the key `release/cosign.pub`; see
[Release evidence](#release-evidence). Compare `release/cosign.pub` with the key published at
<https://github.com/bymorning/releases>.

## Overview

This kit deploys ByMorning on Azure Kubernetes Service in Azure Government or global Azure. Terraform
creates the network, an AKS cluster, PostgreSQL Flexible Server, Blob Storage, Key Vault, and an
optional container registry. Kustomize manifests run the Gateway, one pod that serves the web app and
API, with optional Amazon Bedrock access and a Kubernetes code execution sandbox.

```text
Browser ─► Ingress (TLS) ─► Gateway pod (1 replica, Recreate)
                             ├─► PostgreSQL 17 Flexible Server   delegated subnet, private DNS, TLS
                             ├─► Blob container                  private endpoint, Entra ID only
                             ├─► Key Vault RSA key               private endpoint, wraps stored credentials
                             ├─► Sandbox pods (optional)         code execution, deny-all network policy
                             └─► Amazon Bedrock (optional)       web identity federation or API key
```

```text
environments/example/   Terraform root: provider, module call, outputs
modules/bymorning/      Terraform module
k8s/base/               Namespace, ServiceAccount, Deployment, Service, Ingress
k8s/components/         bedrock-web-identity | bedrock-api-key | code-interpreter-kubernetes
scripts/                overlay.sh, create-secret.sh, fetch-postgres-ca.sh
release/mirror.sh       pushes both release images to your registry and verifies their digests
smoke.sh                post-deployment checks
```

The release directory `release/` holds the two image tarballs, their records (`image-azure.txt`,
`image-execution.txt`), the signing key `cosign.pub`, `mirror.sh`, and the signed evidence folders
`gateway/evidence/` and `code-interpreter/evidence/` (see [Release evidence](#release-evidence)).

Run every command from the directory that contains `installers/` and `release/`.

## Prerequisites

- **Tools:** Azure CLI, Terraform 1.9 or later (`hashicorp/azurerm` 4.81 or later, below 5), `kubectl`,
  `jq`, [crane](https://github.com/google/go-containerregistry/tree/main/cmd/crane), and
  [cosign](https://docs.sigstore.dev/cosign/system_config/installation/).
- **Subscription rights:** Owner, or Contributor plus User Access Administrator. The module creates role assignments.
- **Resource providers registered:** `Microsoft.ContainerService`, `Microsoft.DBforPostgreSQL`,
  `Microsoft.KeyVault`, `Microsoft.Storage`, `Microsoft.ContainerRegistry`, `Microsoft.ManagedIdentity`,
  `Microsoft.Network`, `Microsoft.OperationalInsights`.
- **Quota:** about 8 vCPUs for the defaults (two `Standard_D2s_v5` system nodes, one to three
  `Standard_D4s_v5` user nodes). The optional sandbox pool uses `Standard_D4s_v3`, up to three nodes. Check
  `az vm list-usage -l <region>`; sizes and zones vary by region.
- **Terraform state:** a private, encrypted backend. State contains the database password and cookie key.
  The example root has a commented `azurerm` backend block to start from.
- **Ingress controller and certificate:** the base `Ingress` uses `ingressClassName: nginx`; install
  an NGINX ingress controller, or replace the `Ingress` with your own routing (see Deploy, step 7). You
  need a DNS name for the app and a TLS certificate for it.
- **Identity provider:** a Microsoft Entra tenant (Government tenant for Azure Government) where you
  can register an application, and the email domains your users sign in with.
- **For Amazon Bedrock (optional):** an AWS account, its partition (`aws` or `aws-us-gov`), the Bedrock
  Region, and model access enabled. The cluster must reach AWS STS and Bedrock. Azure Government does not
  imply AWS GovCloud; the partition is your choice.
- **Network access for Terraform:** Key Vault and Storage accept only private endpoints and the
  addresses in `deployer_ip_ranges`. Run Terraform from that address, or from inside the VNet.
  For a private cluster, run `kubectl` from the VNet or use `az aks command invoke`.

### Before you deploy, confirm

- [ ] **Network policy engine.** The module enables Azure CNI Powered by Cilium (`network_policy = "cilium"`).
      Confirm your region supports it. If not, set `network_policy = "azure"` or `"calico"`. The
      code execution sandbox requires an engine that enforces `NetworkPolicy`.
- [ ] **Workload identity audience.** The federated credential uses `api://AzureADTokenExchange` in
      both clouds (`federated_audience`). After deploy, `smoke.sh` prints the token audience; if your
      cloud's webhook injects a different one, set `federated_audience` and apply again.
- [ ] **PostgreSQL private DNS.** Government uses `private.postgres.database.usgovcloudapi.net`;
      confirm that name is valid for your cloud.
- [ ] **Kata pod sandboxing** (only if you enable `sandbox_node_pool`). It needs Azure Linux and a
      nested-virtualization VM size, and cannot be combined with FIPS node pools. Confirm it is available in
      your region.
- [ ] **Private registry access** (if `acr_enabled`). The registry is reachable only from the VNet
      unless `acr_public_network_access = true` (see Configure).

## Configure

```sh
cp -r installers/azure/environments/example installers/azure/environments/<customer>
cd installers/azure/environments/<customer>
cp terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars`. These are the variables the example root exposes:

| Name                       | Required | Example                                  | Meaning                                                                                             |
| -------------------------- | -------- | ---------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `subscription_id`          | Yes      | `"00000000-0000-0000-0000-000000000000"` | Target subscription.                                                                                |
| `environment`              | No       | `"usgovernment"`                         | `"usgovernment"` (default) or `"public"`. Must match the Azure cloud you logged in to.              |
| `location`                 | No       | `"usgovvirginia"`, `"eastus2"`           | Azure region (default `usgovvirginia`).                                                             |
| `deployer_ip_ranges`       | No       | `["203.0.113.10/32"]`                    | Address of the machine running Terraform. Leave empty only when running inside the VNet.            |
| `private_cluster`          | No       | `false`                                  | Private API server (default `true`). When `false`, set `api_authorized_ip_ranges`.                  |
| `api_authorized_ip_ranges` | No       | `["203.0.113.0/24"]`                     | Addresses allowed to reach a public API server. `0.0.0.0/0` is rejected.                            |
| `admin_group_object_ids`   | No       | `["<entra-group-object-id>"]`            | Entra groups granted cluster admin. Turns on Azure RBAC for Kubernetes and disables local accounts. |
| `fips_node_pools`          | No       | `true`                                   | FIPS-enabled node images for the system and user pools.                                             |
| `sandbox_node_pool`        | No       | `true`                                   | Kata VM-isolation node pool for code execution.                                                     |
| `network_policy`           | No       | `"cilium"`                               | `cilium` (default), `azure`, or `calico`.                                                           |
| `acr_enabled`              | No       | `false`                                  | Create an Azure Container Registry (default `true`). Set `false` to use your own registry.          |
| `log_analytics`            | No       | `true`                                   | Create a Log Analytics workspace and enable Container Insights.                                     |
| `federated_audience`       | No       | `"api://AzureADTokenExchange"`           | Audience of the workload identity credential.                                                       |
| `key_vault_sku`            | No       | `"premium"`                              | `standard` (software keys, default) or `premium` (HSM-backed keys).                                 |
| `system_vm_size`, `user_vm_size` | No | `"Standard_D2s_v6"`, `"Standard_D4s_v6"` | Node sizes (defaults `Standard_D2s_v5`, `Standard_D4s_v5`). Pick a family your subscription has quota for; new subscriptions in regions such as `eastus2` may have none for `Dsv5`. |
| `user_node_min`, `user_node_max` | No | `1`, `3` | Autoscaler bounds of the user node pool. |
| `outbound_type` | No | `"userDefinedRouting"` | `loadBalancer` (default), `userDefinedRouting` (your firewall) or `managedNATGateway`. |
| `acr_public_network_access` | No | `true` | Allow pushing to the registry from outside the VNet (default `false`). |
| `postgres_sku` | No | `"GP_Standard_D2ds_v5"` | Database size (default shown). |
| `name`, `tags`             | No       | `name = "bymorning"`                     | Resource name prefix (3 to 16 characters) and tags.                                                 |

Other module settings are not variables in the example root. To change them, add the argument to the
`module "bymorning"` block in `main.tf`. See `modules/bymorning/variables.tf` for all of them; the common ones are:

| Module argument                              | Default                                         | Meaning                                                           |
| -------------------------------------------- | ----------------------------------------------- | ----------------------------------------------------------------- |
| `availability_zones`                         | `[]`                                            | Zones for node pools; leave empty where a region has none.        |
| `postgres_high_availability`                 | `false`                                         | Zone-redundant database HA.                                       |
| `vnet_cidr`, `service_cidr`, `pod_cidr`      | `10.40.0.0/16`, `10.41.0.0/16`, `10.244.0.0/16` | Change if they overlap networks you peer with.                    |

## Deploy

1. **Sign in to Azure.** Use `AzureUSGovernment` for Azure Government, or `AzureCloud` for global Azure.

   ```sh
   az cloud set --name AzureUSGovernment
   az login
   az account set --subscription <subscription-id>
   ```

2. **Create the infrastructure.** In `installers/azure/environments/<customer>`:

   ```sh
   terraform init
   terraform plan
   terraform apply
   terraform output -json | jq 'del(.database_url, .web_cookie_key)' > outputs.json
   ```

   `outputs.json` omits the sensitive values; they reach the cluster through `create-secret.sh`.

3. **Verify and push the images.** ByMorning delivers the Gateway image tarball
   `release/bymorning-a8470c80cd9b.tar` with its record `release/image-azure.txt`, the code execution image
   `release/bymorning-code-interpreter-a8470c80cd9b.tar` with `release/image-execution.txt`, the signing key
   `release/cosign.pub`, and one signed evidence folder per image, `release/gateway/evidence/` and
   `release/code-interpreter/evidence/`. Verify the Gateway image first (`SHA256SUMS` names the tarball as
   `../../bymorning-a8470c80cd9b.tar`, its place in `release/`):

   ```sh
   cd release/gateway/evidence
   sha256sum -c SHA256SUMS                    # macOS: shasum -a 256 -c SHA256SUMS
   cmp cosign.pub ../../cosign.pub            # the evidence key must equal the pinned key
   cosign verify-blob --key ../../cosign.pub --bundle bymorning-*.tar.bundle ../../bymorning-a8470c80cd9b.tar
   cosign verify-blob --key ../../cosign.pub --bundle SHA256SUMS.bundle SHA256SUMS
   cosign verify-blob --key ../../cosign.pub --bundle provenance.intoto.json.bundle provenance.intoto.json
   cd ../../..
   ```

   Each command prints `Verified OK` (or `OK` per file). Repeat in `release/code-interpreter/evidence/` with
   `bymorning-code-interpreter-*.tar.bundle` and `../../bymorning-code-interpreter-a8470c80cd9b.tar`; that folder has no
   provenance statement. Compare `release/cosign.pub` with the key published at <https://github.com/bymorning/releases>.

   Sign in to your registry, then push both tarballs with `release/mirror.sh`. It pushes each OCI layout with `crane`,
   tags it with the release version, fails if a pushed digest differs from the record, and prints the two digest-pinned
   references:

   ```sh
   acr=$(terraform -chdir=installers/azure/environments/<customer> output -raw acr_login_server)
   az acr login --name "${acr%%.*}" --expose-token --output tsv --query accessToken \
     | crane auth login "$acr" --username 00000000-0000-0000-0000-000000000000 --password-stdin
   sh release/mirror.sh "$acr" 1.2.0
   ```

   ```text
   gateway image:   <acr>/bymorning@sha256:<digest>
   execution image: <acr>/bymorning-code-interpreter@sha256:<digest>
   ```

   The registry is private, so run this from the VNet, or set `acr_public_network_access = true`
   in `terraform.tfvars` for the push. If you use your own registry (`acr_enabled = false`), pass its login server
   instead. To push by hand, extract a tarball and use `crane` directly; the digest must equal the `digest:` line of
   its record:

   ```sh
   layout=$(mktemp -d) && tar -xf release/bymorning-a8470c80cd9b.tar -C "$layout"
   crane push "$layout" "$acr/bymorning:1.2.0"
   crane digest "$acr/bymorning:1.2.0"      # must equal the digest in release/image-azure.txt
   ```

4. **Get cluster credentials.**

   ```sh
   az aks get-credentials \
     -g "$(terraform -chdir=installers/azure/environments/<customer> output -raw resource_group)" \
     -n "$(terraform -chdir=installers/azure/environments/<customer> output -raw cluster_name)"
   kubectl apply -f installers/azure/k8s/base/namespace.yaml
   ```

5. **Create the secrets.** `fetch-postgres-ca.sh` downloads the two root CAs Microsoft documents for
   PostgreSQL and prints their fingerprints; compare them with your approved source.

   ```sh
   sh installers/azure/scripts/fetch-postgres-ca.sh > postgres-ca.pem
   sh installers/azure/scripts/create-secret.sh installers/azure/environments/<customer> postgres-ca.pem
   ```

   This creates Secret `bymorning` with `DATABASE_URL`, `WEB_COOKIE_KEY`, and `DATABASE_CA`. Add
   `--extra KEY=<file>` for a secret value from a file (for example `SETUP_TOKEN`), or
   `--literal KEY=VALUE` for a non-secret one.

6. **Generate and apply the overlay.** Use the pinned digest from step 3:

   ```sh
   sh installers/azure/scripts/overlay.sh \
     installers/azure/environments/<customer>/outputs.json <customer> \
     --host bymorning.contoso.us \
     --image "$acr/bymorning@sha256:<digest>"
   kubectl apply -k installers/azure/k8s/overlays/<customer>
   ```

   Add `--component <name>` (repeatable) for the optional components below. The overlay holds no secrets.

7. **Point DNS and TLS at the ingress.** Create a DNS record for the host, and the certificate Secret the `Ingress` names:

   ```sh
   kubectl -n bymorning create secret tls bymorning-tls --cert=<fullchain.pem> --key=<privkey.pem>
   ```

   For Application Gateway for Containers, remove the `Ingress` in your overlay and add a Gateway API
   `HTTPRoute` to Service `bymorning`, port 80. Set `TRUSTED_PROXIES` in the `bymorning-env` ConfigMap
   (base default: the pod CIDR `10.244.0.0/16`) to your ingress controller's addresses so
   `X-Forwarded-For` is trusted for sign-in rate limiting.

8. **Watch the rollout.** The `migrate` init container runs first.

   ```sh
   kubectl -n bymorning rollout status deploy/bymorning
   ```

To keep secrets in Key Vault instead of Kubernetes Secrets, set `csi_secrets_provider = true` in the
module block and mount `postgres-password` and `web-cookie-key` with your own `SecretProviderClass`,
synced to Secret `bymorning`. The kit does not ship that class.

## Code execution

Code runs in short-lived pods, one per call, in namespace `bymorning-sandbox`. The plugin refuses to
execute unless a deny-all `NetworkPolicy` selects the sandbox pods, so the cluster's network policy engine must enforce it.

1. Push the execution image to the registry the sandbox nodes pull from. `release/mirror.sh` in Deploy step 3
   already pushed it (`release/bymorning-code-interpreter-a8470c80cd9b.tar`, recorded in `release/image-execution.txt`,
   verified with `release/code-interpreter/evidence/`) and printed its digest-pinned reference. If you pushed by hand,
   push it as in that step with repository `bymorning-code-interpreter` and compare the digest with the record.
2. Edit `installers/azure/k8s/components/code-interpreter-kubernetes/config/bymorning.jsonc`
   (the shipped `image` is a placeholder; the file is part of the component, so this edit is local to your copy):

   ```jsonc
   "options": {
     "namespace": "bymorning-sandbox",
     "image": "<acr>/bymorning-code-interpreter@sha256:<digest>",   // digest required; a tag is refused
     "serviceAccountName": "bymorning-sandbox",
     "runtimeClassName": "",   // "kata-vm-isolation" for VM isolation
   }
   ```

   For VM isolation, set `sandbox_node_pool = true`, `runtimeClassName` to `kata-vm-isolation`, and
   uncomment `nodeSelector` (`bymorning.dev/pool: sandbox`) and `tolerations`
   (`bymorning.dev/sandbox=true:NoSchedule`) in that file. Otherwise leave `runtimeClassName` empty; sandbox pods
   run as hardened containers on the user pool.

3. Create the sandbox namespace, RBAC, and deny-all policy, then re-run `overlay.sh` with the component and apply:

   ```sh
   kubectl apply -k sandbox
   sh installers/azure/scripts/overlay.sh installers/azure/environments/<customer>/outputs.json <customer> \
     --host bymorning.contoso.us --image "$acr/bymorning@sha256:<digest>" \
     --component code-interpreter-kubernetes
   kubectl apply -k installers/azure/k8s/overlays/<customer>
   ```

   The component mounts the Gateway's service account token so it can reach the Kubernetes API. The
   Role binds `ServiceAccount bymorning-gateway` in namespace `bymorning`; keep those defaults. The sandbox
   manifests, what runs in a sandbox pod, its isolation layers, and every plugin option are described in
   [sandbox/README.md](sandbox.md).

Without a digest-pinned `image`, the Gateway refuses to start with `Azure.ConfigError` (`code_interpreter.image`).
Run `overlay.sh` once with every component you need (repeat `--component`).

## Amazon Bedrock

Choose one. Both need `--aws-region <bedrock-region>`; without the right Region, the STS call goes to `us-east-1`.
For a GovCloud Region (`us-gov-*`) `overlay.sh` also writes `AWS_USE_FIPS_ENDPOINT=true` into the `bymorning-aws`
ConfigMap so Bedrock is dialed through its FIPS endpoint; the image does not set the flag itself, and it is not set at
all for a commercial Region.

**Web identity (no stored AWS secret).** The pod's service account token is exchanged for AWS credentials.

1. Create the IAM OIDC provider in the AWS account, using the AKS issuer. AWS must be able to reach the issuer URL publicly.

   ```sh
   aws iam create-open-id-connect-provider \
     --url "$(terraform -chdir=installers/azure/environments/<customer> output -raw oidc_issuer_url)" \
     --client-id-list sts.amazonaws.com
   ```

2. Create an IAM role with this trust policy. `<provider>` is the provider ARN's path after `oidc-provider/`.

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": { "Federated": "arn:<partition>:iam::<account>:oidc-provider/<provider>" },
         "Action": "sts:AssumeRoleWithWebIdentity",
         "Condition": {
           "StringEquals": {
             "<provider>:aud": "sts.amazonaws.com",
             "<provider>:sub": "system:serviceaccount:bymorning:bymorning-gateway"
           }
         }
       }
     ]
   }
   ```

3. Attach this permission policy. The Gateway calls Bedrock Runtime `ConverseStream`
   (`POST /model/<id>/converse-stream`) for every Bedrock model, which IAM authorizes as
   `bedrock:InvokeModelWithResponseStream`; `Converse` and `ConverseStream` are API operation names, not
   IAM actions, and the Gateway never calls the non-streaming `Converse` or `InvokeModel`. Cross-Region
   inference profiles (IDs such as `us.anthropic.claude-...`) need **both** the inference-profile ARN and
   the foundation-model ARN in every Region the profile routes to
   (`aws bedrock get-inference-profile --inference-profile-identifier <id>` lists them). Missing the second
   produces `AccessDeniedException` from Bedrock.

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": ["bedrock:InvokeModelWithResponseStream"],
         "Resource": [
           "arn:<partition>:bedrock:<region>:<account>:inference-profile/<profile-id>",
           "arn:<partition>:bedrock:<region>::foundation-model/<model-id>"
         ]
       }
     ]
   }
   ```

   Models the catalog serves through Bedrock Mantle (`bedrock-mantle.<region>.api.aws`, `/v1/responses`)
   additionally need `bedrock-mantle:CreateInference` on
   `arn:<partition>:bedrock-mantle:<region>:<account>:project/default`.

4. Regenerate the overlay with the component and apply:

   ```sh
   sh installers/azure/scripts/overlay.sh installers/azure/environments/<customer>/outputs.json <customer> \
     --host bymorning.contoso.us --image "$acr/bymorning@sha256:<digest>" \
     --aws-role-arn arn:<partition>:iam::<account>:role/<name> --aws-region <region> \
     --component bedrock-web-identity
   kubectl apply -k installers/azure/k8s/overlays/<customer>
   ```

**API key.** Create a Bedrock API key in AWS, then store it and use the other component. You own key rotation;
AWS recommends short-term keys for production.

```sh
sh installers/azure/scripts/create-secret.sh installers/azure/environments/<customer> postgres-ca.pem \
  --bedrock-api-key-file ./bedrock-key.txt
sh installers/azure/scripts/overlay.sh ... --aws-region <region> --component bedrock-api-key
```

Do not combine the two components. After either, sign in as an administrator, open **Console → Gateway → Models →
Providers → Add provider**, choose Amazon Bedrock, then **AWS credentials → Use deployment AWS credentials**. The AWS
Region comes from `AWS_REGION`. The session model picker hides models released more than six months ago, so an older Bedrock model
(for example `us.anthropic.claude-haiku-4-5-20251001-v1:0`) is enabled on the Models page but may not be offered in
the picker. Your IAM policy must allow every model you enable. The catalog has no `us.` profile for Nova Micro; use
`amazon.nova-micro-v1:0`.

## First sign-in

1. Read the setup token. The pod's log line `Auth Provider setup pending` names the file; by default:

   ```sh
   kubectl -n bymorning exec deploy/bymorning -c gateway -- cat /tmp/bymorning/setup-token
   ```

   Or choose the token yourself: add `--extra SETUP_TOKEN=<file>` when creating the secret.

2. Open `https://<host>`, go to Setup, and enter the token.
3. Register Entra. In the Azure portal for your cloud (`portal.azure.us` for Government, `portal.azure.com`
   otherwise): **App registrations → New registration**, single tenant, platform **Web**. Under
   **Token configuration**, add the optional ID-token claims `email` and `xms_edov`. Under **Certificates & secrets**,
   create a client secret.
4. In Setup, choose **Microsoft Entra**, select the cloud, and enter the tenant ID, application (client)
   ID, client secret, and the email domains allowed to sign in. Setup shows the redirect URI,
   `https://<host>/auth/callback/<providerID>`. Add it to the app registration as a Web redirect URI, then sign in.

Entra sends `xms_edov` instead of `email_verified`. ByMorning admits a user whose ID token has an `email` in an
admitted domain with `xms_edov: true`, so verify each admitted domain under **Custom domain names**.

Later changes to the identity provider are made under Settings → Security.

**Alternative: seed the provider at deploy time.** Set all five values together, or startup fails. Register the
redirect URI `https://<host>/auth/callback` (no provider ID) for a provider seeded this way:

```sh
sh installers/azure/scripts/create-secret.sh installers/azure/environments/<customer> postgres-ca.pem \
  --literal AUTH_PROVIDER=microsoft \
  --literal AUTH_ISSUER=https://login.microsoftonline.us/<tenant-id>/v2.0 \
  --literal AUTH_CLIENT_ID=<client-id> \
  --literal AUTH_EMAIL_DOMAINS=contoso.us \
  --extra AUTH_CLIENT_SECRET=./client-secret.txt
```

Use `https://login.microsoftonline.com/<tenant-id>/v2.0` for a global Azure tenant.

## Verify

```sh
sh installers/azure/smoke.sh --origin https://bymorning.contoso.us
```

By default `smoke.sh` only reads: replica count and strategy, the `migrate` init container, environment,
`/system/health`, `/auth/method`, and the sign-in page headers. Options:

| Option                       | Adds                                                                                                                                                                                                                                                        |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--temp --cli-image <image>` | Short-lived pods and a blob: workload identity, Blob upload and readback, Key Vault wrap and unwrap, sandbox RBAC and deny-all egress, and the AWS `AssumeRoleWithWebIdentity` call. Mirror `mcr.microsoft.com/azure-cli` into your registry for the image. |
| `--cloud AzureCloud`         | Use for global Azure (default is `AzureUSGovernment`).                                                                                                                                                                                                      |
| `--rotate`                   | Rotates the Key Vault key and restarts the Gateway.                                                                                                                                                                                                         |
| `--code-cleanup`             | After you ran code in the UI, checks that no sandbox pods remain.                                                                                                                                                                                           |

It prints manual steps for what needs a signed-in user: a chat and a code execution.

## Upgrade

1. Verify the new release and push its images with `release/mirror.sh` as in Deploy step 3.
2. Take a database backup (see Rollback).
3. Re-run `overlay.sh` with the new digest and the same options, then apply:

   ```sh
   kubectl apply -k installers/azure/k8s/overlays/<customer>
   ```

The pod is replaced (one replica, `Recreate`), so expect a short outage. Migrations run first in the `migrate`
init container. For a Terraform change, `terraform plan` then `terraform apply` in the environment. After
rotating the Key Vault key, restart the Gateway; it resolves the latest key version at start. `smoke.sh --rotate`
does both; the Ingress may answer 503 for a few seconds after the new pod is Ready, which `smoke.sh` waits out.

**Upgrading to 1.2.0.** 1.2.0 ships one Gateway image for every cloud; the manifests bind it to Azure. The base
`bymorning-env` ConfigMap gains two keys, `STORAGE_PROVIDER=azure-blob` and `CIPHER_BACKEND=keyvault`, so apply the
1.2.0 `k8s/` tree together with the 1.2.0 image: re-run `overlay.sh` from this release's `installers/azure` and
`kubectl apply -k` once. The 1.2.0 image refuses to start without the selectors and logs the name of the missing
variable; the 1.1 image ignores them. `AWS_USE_FIPS_ENDPOINT` is no longer baked into the image: if you run Bedrock in
a GovCloud Region, the regenerated overlay writes it into `bymorning-aws` (see Amazon Bedrock).

## Rollback

- **Gateway version:** re-apply the overlay with the previous digest. Migrations only move forward, so an older
  image against a migrated database is unsupported. To undo a migration, restore the database with
  point-in-time restore (14-day backups by default) before rolling back.
- **Key rotation:** nothing to undo. Old key versions stay enabled and decrypt old data; never disable or delete them.

## Uninstall

1. Export anything you need from the Blob container and the database.
2. `kubectl delete -k installers/azure/k8s/overlays/<customer>`, and `kubectl delete -k sandbox` if you deployed the sandbox.
3. `terraform destroy` in the environment.

Key Vault has purge protection with 90-day soft delete: the vault and its key are kept for 90 days and cannot
be purged sooner. The example root also refuses to delete a resource group that still contains resources; remove
the remaining resources first. Do not recreate the key in an existing vault.

## Troubleshooting

**Pods stay in `ImagePullBackOff`.** The image reference or registry access is wrong. Confirm the overlay's image
and that the kubelet has `AcrPull` (`acr_pull` defaults to `true`), and for a private registry that the node subnet can reach
its private endpoint. `kubectl -n bymorning describe pod` shows the pull error.

**AKS refuses a VM size (`The VM size of ... is not allowed`) or quota is zero.** New subscriptions can have zero
quota for the default `Dsv5` family. Check `az vm list-usage -l <region>` and set `system_vm_size` and `user_vm_size`
(root variables) to a family with quota that the AKS error lists (for example `Standard_D2s_v6` and `Standard_D4s_v6`).

**`terraform apply` cannot create the Key Vault key or Storage container.** The Terraform runner has no
data-plane access. Set `deployer_ip_ranges` to the runner's address (single hosts are written without `/32`), or run inside the VNet.

**The pod crashes at startup.** `kubectl -n bymorning logs deploy/bymorning -c gateway --previous`.
`Host.ConfigError` names the setting: a Storage or Key Vault host from the other cloud, a Microsoft issuer from the other cloud, or a missing code
execution image. A failed `migrate` init container shows in `-c migrate`.

**Blob or Key Vault calls are denied.** The workload identity token audience must equal `federated_audience`. Run
`smoke.sh --temp` (it prints the token audience) and set `federated_audience` to the value shown, then apply again.

**Sign-in fails with an email-domain refusal.** The token needs `email` and `xms_edov: true`, and the user's email domain
must be verified in the tenant and admitted in ByMorning. Entra guest (`#EXT#`) users and users without a `mail`
attribute are refused. A browser already signed in to another Microsoft account signs in as that account; use a private window.

**Redirect URI mismatch (`AADSTS50011`).** Register exactly the URI Setup shows (with the provider ID); a provider seeded from `AUTH_*` uses `/auth/callback`.

**Bedrock calls fail with `AccessDeniedException`.** The role policy needs `bedrock:InvokeModelWithResponseStream` on both the inference-profile ARN and the
foundation-model ARN(s); see [Amazon Bedrock](#amazon-bedrock). Also check that model access is enabled and that `AWS_REGION` is the Bedrock Region.
If token exchange fails, check the OIDC provider, the role's `sub` (`system:serviceaccount:bymorning:bymorning-gateway`), and that AWS can reach the issuer URL.

**Code execution fails to start or refuses to run.** The sandbox namespace needs the deny-all `NetworkPolicy`, and the cluster needs an
engine that enforces it. Confirm `kubectl -n bymorning-sandbox get networkpolicy` and that Kata pods use the node selector and toleration.

## Reference

### Outputs

| Output                                                             | Use                                                     |
| ------------------------------------------------------------------ | ------------------------------------------------------- |
| `resource_group`, `cluster_name`                                   | `az aks get-credentials`                                |
| `oidc_issuer_url`                                                  | AWS OIDC provider for Bedrock web identity              |
| `azure_cloud`, `tenant_id`, `gateway_client_id`                    | Overlay values (`overlay.sh` reads them)                |
| `federated_audience`                                               | Audience of the workload identity credential            |
| `namespace`, `service_account`                                     | Kubernetes namespace and service account of the Gateway |
| `storage_account_url`, `storage_account_name`, `storage_container` | Blob storage                                            |
| `key_vault_name`, `key_vault_key`                                  | Key Vault and its versionless key URL                   |
| `postgres_fqdn`                                                    | PostgreSQL server                                       |
| `acr_login_server`                                                 | Registry (`null` when `acr_enabled = false`)            |
| `database_url`, `web_cookie_key`                                   | Sensitive; read by `create-secret.sh`                   |

### Runtime settings

The Gateway image is the same for every cloud; the `bymorning-env` ConfigMap binds it to Azure. The base sets the
two selectors, `STORAGE_PROVIDER=azure-blob` and `CIPHER_BACKEND=keyvault` (required; the image refuses to start
without them, and they are fixed for the life of an installation), plus `BYMORNING_CONFIG=/config/bymorning.jsonc`
(the installation document from the `bymorning-config` ConfigMap) and `TRUSTED_PROXIES`. The overlay merges
`APP_ORIGIN`, `AZURE_CLOUD` (`AzureCloud` or `AzureUSGovernment`), `STORAGE_ACCOUNT_URL`, `STORAGE_CONTAINER`, and
`KEY_VAULT_KEY` from the Terraform outputs. `AWS_REGION` here means the Bedrock Region and lives in `bymorning-aws`
with the other Bedrock settings (see Amazon Bedrock). Optional settings you can add as ConfigMap literals or Secret
keys:

| Name                  | Meaning                                                                                               |
| --------------------- | ----------------------------------------------------------------------------------------------------- |
| `SETUP_TOKEN`         | Setup token you choose instead of a generated one                                                     |
| `LOGIN_BANNER`        | System use notification acknowledged before sign-in                                                   |
| `TLS_CERT`, `TLS_KEY` | PEM listener certificate and key (both or neither). Then switch probes and backend protocol to HTTPS. |

### Release evidence

Both images are built from one source commit (the `# built from` line of `release/image-*.txt`) and signed with the
ByMorning release key `release/cosign.pub` (ECDSA P-256, held in a hardware-backed KMS). Each evidence folder has a
signed `SHA256SUMS` covering its tarball (as `../../<tarball>`) and every report beside it.

| Folder                               | Contents                                                                                                                                                                                                                                                                                                                              |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `release/gateway/evidence/`          | Gateway image: image SBOMs (`sbom-image.cyclonedx.json`, `sbom-image.spdx.json`), source SBOM (`sbom-source.cyclonedx.json`), `vulnerabilities-image-grype.json`, `vulnerabilities-source-grype.json`, `vulnerabilities.md`, `licenses.md`, `gitleaks.json`, `semgrep.sarif`, `bun-audit.txt`, `openvex.json`, `provenance.intoto.json` (SLSA v1), `tools.txt`, and the `.bundle` signatures |
| `release/gateway/`                   | `vulnerability-triage.md`: the ByMorning Security position on every Critical and High finding                                                                                                                                                                                                                                       |
| `release/code-interpreter/evidence/` | Code execution image: image SBOMs, `vulnerabilities-image-grype.json`, `vulnerabilities.md`, `tools.txt`, and the `.bundle` signatures                                                                                                                                                                                              |

### Design notes

- One replica and `Recreate`: OAuth callback state and the scheduler are process-local, and two versions must never overlap.
- The Gateway uses the PostgreSQL administrator login. A least-privilege application role needs a step inside the VNet and is not automated.
- Key Vault keys rotate at 18 months and expire at 2 years (`key_rotate_after`, `key_expire_after`). Expired versions still unwrap but cannot wrap.
- Tool output under `tool-output/` expires after 7 days (`tool_output_days`).
- A FIPS node pool covers node cryptography. The Gateway image runs on the RHEL 9 OpenSSL FIPS Provider regardless of the node.
