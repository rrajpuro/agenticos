# OpenSandbox workspaces: file persistence, outbound access and credentials

**Proposal, 2026-11-23.** Keep the files, replace the compute, and authorize
credentials for a run rather than for a directory. These are three separate
features. The current integration does not provide these guarantees; the
evidence below is repository and upstream source review, not a live test.

## What should survive?

Suppose an agent writes `/workspace/report.md` in a project. A week later the
sandbox has been deleted. The next authorized agent in that project should see
the report in a fresh sandbox. It should **not** inherit the previous process,
installed system packages, network policy or credentials.

| Today | Where the files are | What happens when compute goes away |
| --- | --- | --- |
| [sandboxd](../sandbox.md#how-long-anything-survives) | A directory on the sandboxd host is mounted at `/workspace`. The Files panel can read it without starting a container. | Idle compute can be reaped while the directory stays. Workspace retention and explicit deletion are separate. |
| [OpenSandbox adapter](../../backend/app/agents/capabilities/sandbox/_opensandbox.py) | AgenticOS requests an image but no workspace volume. Persistent scopes record a reference to the remote sandbox, not a copy of the files. | An idle but intact sandbox can be reattached. If unavailable, refuse rather than silently replace it. If destroyed, its private filesystem is not recoverable from the reference. |

**Delivery plan:** (1) opt-in durable files for new project and conversation
workspaces; (2) connection-level outbound policy, usable with or without durable
files; (3) outbound credentials for private, run-scoped compute only. Storage
and egress can ship independently. Credentials require verified egress and
crash-safe cleanup, **not** durable storage. Existing connections, published
specs and recorded sandbox references retain their behavior unless explicitly
migrated. This is for OpenSandbox agent workspaces, not sandboxd workflow code,
browser tools or HTTP calls made by the AgenticOS backend.

## 1. Make the workspace independent of the sandbox

**Use storage on the OpenSandbox side by default.** Mount one project or
conversation volume at `/workspace`. On Docker this is a named volume on the
selected daemon; on Kubernetes it is a persistent volume claim (PVC) in the
sandbox namespace. A path on the AgenticOS API server is not a path on a
remote worker. If the files must reside on infrastructure shared with
AgenticOS, the operator can back a PVC with shared storage such as NFS. The API
server need not mount it. Do not expose host paths, claim names or arbitrary
mount definitions to agents. See the [Docker][docker-volume] and
[Kubernetes][k8s-volume] examples.

**Ownership.** Give an eligible connection an opt-in storage profile chosen by
the operator. The [workspace service](../../backend/app/services/sandbox_workspace.py)
resolves the effective scope (including project/exposure overrides) and owns two
distinct records:

| Record | What it identifies | Lifetime |
| --- | --- | --- |
| Workspace storage | Organization, owning project or conversation workspace, generation, approved storage location and provider volume identity | Until the owning workspace is explicitly deleted, moved or reaches its configured retention limit |
| Compute attachment | Current sandbox ID, connection endpoint revision, storage generation, applied policy revision and writer lease | Until the sandbox stops or is retired; several successive sandboxes may use one volume |

Use an opaque, server-allocated volume identity; verify ownership and location
before mounting or deleting it. Don't adopt a pre-existing volume just because
its name matches a reservation. Never take the volume name from the prompt,
agent spec or caller. A rename, image change or connection edit must not silently
select a different volume. A change of storage location requires a move.
Run-scoped scratch remains disposable; other scopes stay on their existing path
until they have an explicit ownership and retention contract.

**Open and close.** Reserve the storage generation durably before first
provisioning. Only that initial allocation may create a volume; reconcile an
ambiguous result against the reservation. For all subsequent mounts require
the recorded volume to exist (`createIfNotExists=false`). A missing or
unverifiable volume is a recovery error, not consent to create an empty one.
Keep `deleteOnSandboxTermination=false`. Set the mount on sandbox creation and
verify it before handing the workspace to tools.

Serialize writer acquisition across all runs using that generation. Reattach
only to the matching sandbox and policy. Before mounting a replacement, prove
the old writer is stopped or fence it from storage; an attach timeout or an
expired database lease alone does neither. If the provider cannot confirm that
condition, wait or report the workspace unavailable. Idle retirement releases
compute, not the volume. Files outside `/workspace` are not promised to survive.

**Move, migrate and delete.** Never mount a new volume over a legacy sandbox's
only copy of its files. Reuse the [project transfer](../../backend/app/services/project_workspace_transfer.py)
pattern: quiesce writers, copy into a new generation, verify the file tree and
digests, then switch the active reference. Leave the source intact on failure.
Give the transfer service a source-only, bounded file export when the old
policy no longer permits normal opens. Do not return a shell or provision
credentials on that path. If safe export isn't available, report the blocker;
don't offer a move that requires bypassing the policy or a reset that loses files.

Deleting the owning workspace must queue **volume** cleanup even when no sandbox
ID remains. Commit the deletion intent with the ownership change, detach old
compute first, and retry against the original location and generation, never
the current target of an edited connection. Use a storage controller with
scoped provision/inspect/delete authority; the pinned [lifecycle API][lifecycle]
has no stand-alone volume deletion endpoint. Keep the feature off on deployments
without a storage owner that can complete that cleanup. Verify reclaim behavior, quota
and retention with the operator. A persistent volume is not a backup.

**Scope limit:** this delivers files to the agent across sandbox replacement.
It does not deliver sandboxd's offline Files panel or archive downloads. Those
would need a separately authorized file-reading service when no compute exists;
see [current OpenSandbox behavior](../sandbox.md).

## 2. Apply outbound policy to new compute

Add typed settings under **Settings → Sandboxes → Connections**, only for
OpenSandbox. Keep SDK models inside the adapter, not in agent specs. Present
these as *requested sandbox rules*, subject to deployment-wide rules:

| Setting | Creation request |
| --- | --- |
| Server defaults (existing behavior) | Omit policy. On classic deployments that can mean no egress sidecar, not a platform-enforced baseline. |
| Deny by default, no exceptions | Default deny, empty rules |
| Domain allowlist | Default deny with exact allow rules |
| Domain blocklist | Default allow with exact deny rules |

Accept normalized domain names only: no schemes, paths, ports, IPs, CIDRs or
wildcards. Explain the limit in the UI: DNS decides which names resolve;
in `dns+nft` mode packet filtering also admits resolved **addresses** for a
limited time. Two hostnames can share an address. This is not per-request
hostname isolation. Operator exceptions and port restrictions also affect the
result. A credential match, by contrast, checks an individual HTTPS request.
See [egress architecture][egress] and [operator rules][network].

Record the applied revision on each compute attachment. A change governs new
work, not already running processes. Refuse normal use of a stale attachment,
including recorded-ref operations; don't redirect an operation intended for an
old sandbox to a replacement. For durable files, retire the old writer before
starting new compute against the same volume. For legacy files, require an
explicit verified move or destructive reset. Refs with no revision remain
eligible only for the unchanged legacy default. Do not patch an old policy
in place: SDK rule edits do not replace its default action.

## 3. Grant outbound credentials to a private run

There are two different keys. The **connection key** lets AgenticOS talk to
OpenSandbox. The **destination credential** lets a matching request from a
sandbox call an approved upstream. Never put the latter in the image, mount,
command line, environment, spec, response, audit payload or logs.

- **Grant:** A principal authorized to use the specific organization
  [HTTP credential](../secrets.md) and manage the agent binding approves the
  agent, allowed run context, connection and control-plane endpoint revision.
  The spec references that binding, never an arbitrary secret/URL pair. A
  connection UUID is insufficient: its server address can change. Invalidate
  approval on endpoint change, including when editing the same connection.
  Use the same approved endpoint for the write; never follow a redirect or
  re-resolve a changed default to deliver plaintext elsewhere.
- **Destination:** Require an exact HTTPS origin on port 443 accepted by the
  sealed credential's [origin check](../../backend/app/core/secret_kinds.py),
  an explicit domain allow rule, methods, narrow paths, and bearer or named
  API-key header auth. Reject conflicting matches. No automatic host additions,
  wildcard grants, personal credentials, substitutions or token exchanges.
  A trusted upstream may still return sensitive data in its response.
- **Run:** Validate at publish; recheck the actual principal, exposure, agent,
  grant, effective scope, endpoint and policy before opening tools or unsealing
  the credential. [Vault lookup](../../backend/app/services/organization_secret.py)
  is not authorization. Don't inherit the publisher's authority for an
  unapproved caller. First release: private run-scoped scratch only, **no
  shared durable mounts and no delegate handoff**. Reject incompatible sharing
  at publish and guard the [runtime handoff](../../backend/app/services/agent_runner.py)
  too, including nested delegates: they otherwise receive the already-open
  backend without another workspace open.

**Provisioning order matters.** Run scope today has no persistent workspace row,
and normal release destroys through an in-memory ref. Add a committed run-owned
cleanup lease, not just an exception handler:

1. Reserve an attempt with the approved endpoint and recovery identity before
   creation. Create once with a finite provider lifetime, explicit default-deny
   policy and credential proxy. Reconcile ambiguous creation; don't retry it
   blindly or let tools use an unverified sandbox.
2. Commit the returned sandbox ID to the lease **before** the vault write. If
   that commit fails, do not send the secret. Verify the policy, populate the
   sidecar from the trusted control plane, then verify sanitized binding
   metadata before making any tool available.
3. On completion, cancellation, failed or ambiguous provisioning, or loss of
   the owning worker, destroy that sandbox. A recovery worker claims abandoned
   leases, retries cleanup, and clears the record only on confirmed absence.
   Don't unseal the destination credential to clean up. Preserve the original
   teardown route while a lease exists.

Grant revocation stops new admissions. To end an active grant, terminate its
sandbox; for emergency revocation, revoke the upstream token too. The classic
sidecar vault is in memory: Kubernetes pause/resume loses it, Docker unpause
preserves it only while the sidecar survives. Don't resume a credentialed run
on an empty vault or silently inject its credential into shared compute.
See the [Credential Vault lifecycle][vault].

## Implementation and release gates

The [locked dependencies](../../backend/uv.lock) and public APIs set two
different implementation gates:

- [OpenSandbox SDK 1.2.1][sdk] supports creation-time volumes, network policy
  and credential proxy. The separate post-create vault write installs bindings.
  [Volume and policy models][models] describe the provider contract.
- [Harness 0.68.0][harness-068] owns creation, recovery tagging and cancellation;
  it exposes neither volume forwarding nor the two outbound arguments.
  [Harness 0.69.0][harness-069] forwards `network_policy` and `credential_proxy`
  on the lower-level backend, **not volumes**. It requires the coordinated
  [Pydantic AI 2.69.0 upgrade][pyd-269]. Leave `allow_internet_access` at its
  default with an explicit policy. Use `get_sandbox()` before exposing tools.
- For storage, obtain a supported public volume/create hook and verify recovery
  and cancellation still work. Do not fork the private creation method or assume
  the outbound upgrade unlocks mounts. Until then, keep durable storage off.
  Keep lazy creation and no-retry behavior for unchanged connections.

Enable features independently per verified connection. A shell probe or
[existing Secure Access validation](../kubernetes.md#oidc-and-sandbox-configuration)
demonstrates neither volume retention nor outbound enforcement; Secure Access
protects inbound endpoints. Pin the tested server, controller/execd and egress
images. For the outbound phases, configure [server egress][server] with an
egress image and `dns+nft`; DNS-only mode is insufficient. Validate Docker
bridge networking or Kubernetes nftables and sidecar `NET_ADMIN` as applicable.
Exclude unsupported gVisor, transparent mesh interception and Fast Sandbox
profiles from the initial sidecar rollout. Verify interception CA trust,
upstream TLS verification and authenticated, encrypted vault delivery.

| Gate | Evidence required before enabling |
| --- | --- |
| Durable storage | On each supported runtime, write a file, destroy the sandbox, create a different sandbox, read identical bytes. An absent volume fails closed. A second tenant cannot select it. An unreachable or concurrent writer cannot cause two writable attachments. Deleting a conversation/project with no live compute purges only its volume. |
| Migration | Copy a stale-policy legacy workspace without agent execution. Interrupt before cutover: source is unchanged. Complete cutover: destination matches the source and a delayed cleanup cannot delete it. |
| Outbound policy | On the target deployment, check an allowed name, a denied name and a disallowed address with no lease. Document shared-address limitations. Stale refs cannot run new work and changing a policy does not discard files. |
| Credentials | Use only a throwaway token and controlled HTTPS endpoint. Only the approved host/method/path receives it. Cross-tenant grants, endpoint swaps, revoked grants, persistent scope overrides and parent-to-child sharing (including nested delegates) fail before handoff. No secret in logs or metadata. |
| Cleanup | Stop the worker immediately after a successful vault write; separately fail normal destruction and time out provisioning. After restart, confirm teardown on the original server and no second sandbox creation. |

If a gate fails, stop new admissions for that feature; keep cleanup and access
to recovery records working. Disabling durable storage must not delete volumes
or reinterpret existing workspaces as empty scratch. Report unverified features
as such; do not infer support from a version string.

[lifecycle]: https://github.com/opensandbox-group/OpenSandbox/blob/release-1.2.1/specs/sandbox-lifecycle.yml
[docker-volume]: https://github.com/opensandbox-group/OpenSandbox/blob/release-1.2.1/docs/examples/docker-pvc-volume-mount.md
[k8s-volume]: https://github.com/opensandbox-group/OpenSandbox/blob/release-1.2.1/docs/examples/kubernetes-pvc-volume-mount.md
[sdk]: https://github.com/opensandbox-group/OpenSandbox/blob/release-1.2.1/sdks/sandbox/python/src/opensandbox/sandbox.py
[models]: https://github.com/opensandbox-group/OpenSandbox/blob/release-1.2.1/sdks/sandbox/python/src/opensandbox/models/sandboxes.py
[vault]: https://github.com/opensandbox-group/OpenSandbox/blob/release-1.2.1/docs/guides/credential-vault.md
[harness-068]: https://pypi.org/project/pydantic-ai-harness/0.68.0/#files
[harness-069]: https://pypi.org/project/pydantic-ai-harness/0.69.0/#files
[pyd-269]: https://github.com/pydantic/pydantic-ai/releases/tag/v2.69.0
[server]: https://github.com/opensandbox-group/OpenSandbox/blob/release-1.2.1/server/configuration.md#egress
[network]: https://github.com/opensandbox-group/OpenSandbox/blob/release-1.2.1/docs/guides/network-security.md
[egress]: https://github.com/opensandbox-group/OpenSandbox/blob/release-1.2.1/docs/architecture/network/egress.md
