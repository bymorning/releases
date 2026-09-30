# Code execution sandbox

Code Interpreter runs one command or source program per call in a fresh pod in the dedicated namespace
`bymorning-sandbox`. The pod is created for the call and deleted when the call ends, however it ends. No state is
kept between calls. The Gateway creates and execs into these pods through the Kubernetes API.

Installation steps are in [the installation guide](README.md#code-execution). This folder holds the Kubernetes
objects the sandbox needs:

```sh
kubectl apply -k sandbox
```

| File                  | Purpose                                                                                                  |
| --------------------- | -------------------------------------------------------------------------------------------------------- |
| `namespace.yaml`      | `bymorning-sandbox` with Pod Security Admission `restricted` (enforce, audit, warn)                      |
| `serviceaccount.yaml` | Unbound `bymorning-sandbox` service account with token automount off                                     |
| `rbac.yaml`           | Namespaced Role and RoleBinding for the Gateway's service account (`bymorning-gateway` in `bymorning`)   |
| `network-policy.yaml` | Deny all ingress and egress for every pod in the namespace                                               |
| `resource-quota.yaml` | Optional ResourceQuota (10 pods) and LimitRange; listed in `kustomization.yaml`, remove it there to omit |

If the Gateway's service account or namespace differ from the defaults, edit the RoleBinding subject first.

## Permissions

The Role is namespaced (no ClusterRole):

```yaml
rules:
  - { apiGroups: [""], resources: ["pods"], verbs: ["create", "get", "list", "delete"] }
  - { apiGroups: [""], resources: ["pods/exec"], verbs: ["get", "create"] }
  - { apiGroups: ["networking.k8s.io"], resources: ["networkpolicies"], verbs: ["get", "list"] } # only for requireNetworkPolicy
```

The Gateway identity can exec into any pod in the sandbox namespace, so the namespace must hold nothing else.

## What runs, and its limits

| Aspect            | Behavior                                                                                                                |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Input             | `command` runs `/bin/sh -c`; `code` runs Python under `python3`, JavaScript and TypeScript under `bun`                  |
| Output            | `{ stdout, stderr, exitCode }`. A non-zero exit is output, not an error. Limited to 100,000 characters combined         |
| Working directory | `/work` (memory-backed, 64 MiB) and `/tmp` (64 MiB). The root filesystem is read-only                                   |
| Timeouts          | The program is limited to `timeoutSeconds` (default 120). The kubelet also kills the pod after the start and run limits |
| Orphans           | Pods left by a Gateway restart are swept on the next activation and periodically, by label                              |

## Isolation

| Layer                                                   | Provides                                                                                                                                             | Does not provide                                                                                                                  |
| ------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| Pod spec                                                | Non-root UID, read-only root, all capabilities dropped, no privilege escalation, seccomp `RuntimeDefault`, no service account token, resource limits | A PID limit (a node-pool kubelet setting) or protection against a kernel or runtime escape                                        |
| Pod Security Admission `restricted`                     | The API server rejects any pod that is not restricted                                                                                                | Anything beyond pod-spec policy                                                                                                   |
| Deny-all NetworkPolicy                                  | Blocks ingress and egress: DNS, the pod network, the API server, cloud metadata endpoints                                                            | Enforcement is done by the cluster's network policy engine. If it does not enforce `NetworkPolicy`, pods have full network access |
| Startup checks (`requireNetworkPolicy`, `verifyEgress`) | Refuse to run when no deny-all policy selects the pods or a pod can still reach the API server                                                       | Proof of enforcement for every destination                                                                                        |
| `runtimeClassName` (for example `kata-vm-isolation`)    | A VM boundary per pod, when the runtime is installed                                                                                                 | Availability in your region, which you confirm. Off by default                                                                    |
| Ordinary containers                                     | Namespace, cgroup, seccomp, and capability isolation of a hardened container                                                                         | VM-grade isolation. All sandbox pods share the node kernel                                                                        |

A pod can be Ready before its NetworkPolicy is enforced, so each pod verifies that egress is blocked before code runs
and fails closed otherwise.

The execution image contains no ByMorning secrets or credentials. **No FIPS or other compliance claim is made for code
that runs inside it.** A FIPS-enabled node pool covers node kernel cryptography only, not container userland.

## Options

Set these under `options` of the plugin entry in
`installers/azure/k8s/components/code-interpreter-kubernetes/config/bymorning.jsonc`. Non-secret values only.

| Option                              | Default               | Meaning                                                                             |
| ----------------------------------- | --------------------- | ----------------------------------------------------------------------------------- |
| `image`                             | required              | Execution image, `name@sha256:<64 hex>`. A tag is refused                           |
| `imagePullPolicy`                   | `IfNotPresent`        | `IfNotPresent`, `Always` or `Never`                                                 |
| `imagePullSecret`                   | none                  | Name of an image pull Secret in the sandbox namespace                               |
| `namespace`                         | `bymorning-sandbox`   | Dedicated sandbox namespace (never the Gateway's)                                   |
| `serviceAccountName`                | `bymorning-sandbox`   | Unbound service account for sandbox pods. Its token is never mounted                |
| `runtimeClassName`                  | none                  | For example `kata-vm-isolation`                                                     |
| `nodeSelector`                      | none                  | Map of node labels                                                                  |
| `tolerations`                       | none                  | Kubernetes tolerations                                                              |
| `installation`                      | `default`             | Label scoping the orphan sweep. Give each Gateway sharing a namespace its own value |
| `runAsUser`                         | `1000`                | Numeric user; must match the image                                                  |
| `cpu`, `memory`, `ephemeralStorage` | `1`, `512Mi`, `256Mi` | Requests and limits (equal)                                                         |
| `scratchSize`                       | `64Mi`                | Size of the memory-backed `/work` and `/tmp`. They count toward the memory limit    |
| `startTimeoutSeconds`               | `60` (5 to 600)       | Wait for the pod to run and for the egress check                                    |
| `timeoutSeconds`                    | `120` (1 to 3600)     | Program run time                                                                    |
| `requireNetworkPolicy`              | `true`                | Refuse to run unless a deny-all NetworkPolicy selects the pods                      |
| `verifyEgress`                      | `true`                | Confirm egress is blocked inside each pod before running code                       |
