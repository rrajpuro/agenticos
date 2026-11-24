# OpenSandbox workspaces: persistent files, browsing and outbound access

**Proposal, 2026-11-24.** An agent writes `/workspace/report.md`. A conversation
participant should be able to download the report from Files while the sandbox
is running. If the sandbox is deleted but the workspace is retained, the same
participant should still be able to download it, and the next authorized run
should see the same file in a fresh sandbox. Neither the old process nor its
credentials should survive. This is a design, not a tested deployment.

## What works today, and what changes

| Operation | Today | Proposed change |
| --- | --- | --- |
| Continue in the same sandbox | AgenticOS can reattach to an intact OpenSandbox sandbox by its recorded ID. | None. Reattachment does not need a volume or change existing mounts. |
| Keep files across sandbox replacement | The [adapter][adapter] requests an image but no workspace volume. The sandbox ID is not a copy of its files. | Mount a retained volume when creating a sandbox for an opted-in project or conversation. |
| Browse workspace files from the console | [sandboxd](../sandbox.md#how-long-anything-survives) can serve files without running the agent container. Agent tools can use OpenSandbox files, but the console does not browse them. | Use the existing Files UI and authorized routes, with separate readers for a live sandbox and a retained volume. |
| Control outbound requests | The adapter does not pass a network policy or credential proxy. | Add connection-level network rules and, independently of storage, destination credentials for private runs. |

## Keep workspace files when compute is replaced

[Pydantic AI Harness PR #14282][volumes-pr] is the upstream dependency. As of
November 24 it is open: it adds `OpenSandboxBackend(volumes=...)` and forwards
SDK `Volume` objects to `Sandbox.create()`. Omitting volumes leaves the creation
request unchanged. It does not retrofit mounts on reattach, manage storage or
provide a Files panel. Enable durable storage only with a released Harness
version that includes this support.

**Storage stays on the OpenSandbox side.** Mount one [Docker named volume][docker-volume]
or [Kubernetes PVC][k8s-volume] at `/workspace`. The AgenticOS API server does
not mount it. An operator enables a storage profile on a connection; the
[workspace service](../../backend/app/services/sandbox_workspace.py) resolves
the effective project or conversation, including scope overrides. Run-scoped
scratch stays disposable. Existing workspaces keep their behavior unless
explicitly migrated.

| Record | What AgenticOS keeps |
| --- | --- |
| Workspace storage | Organization, owning project or conversation, storage location, opaque provider volume ID and generation. Retain until owner deletion, migration or retention expiry. |
| Compute attachment | Sandbox ID, connection endpoint revision, storage generation, applied policy revision and writer lease. Retiring compute does not retire storage. |

- **Allocate once.** Reserve a generation before provisioning. Reconcile an
  ambiguous create against that reservation; never adopt a volume by name
  alone. Later mounts must require the recorded volume to exist
  (`createIfNotExists=false`); keep `deleteOnSandboxTermination=false`. Agents
  and callers cannot supply host paths, claim names or volume IDs.
- **Replace safely.** Verify the mount and serialize writers. Before mounting
  a replacement, confirm the previous writer stopped or prevent it from writing
  to the volume. An expired lease or unreachable sandbox is not proof. If the
  volume or writer cannot be verified, report the workspace unavailable rather
  than serving empty storage.
- **Move explicitly.** Never mount over the only copy of legacy files. Follow
  the [project transfer](../../backend/app/services/project_workspace_transfer.py)
  pattern: quiesce writers, copy, verify file digests, then switch generations.
  Leave the source intact on failure. If policy blocks a normal open, allow
  only a bounded source export without a shell or credentials; otherwise
  report that the move is blocked.
- **Delete by ownership.** Queue cleanup with the ownership change, even when
  no sandbox ID remains. Detach users of the volume, including file readers,
  then retry deletion against its original location and generation. An edited
  connection must never redirect cleanup. The [OpenSandbox lifecycle API][lifecycle]
  has no standalone volume deletion operation; enable retention only where a
  storage controller can verify ownership, reclaim storage and enforce quota.

## Browse files from AgenticOS

Reuse the existing conversation and project file routes for listing, text
preview and byte downloads. Keep folder ZIP downloads where the route already
supports them. For conversation files, preserve conversation read access,
including shares; for project files, preserve project view access. Check each
request, not just the listing. AgenticOS selects the reader from its records;
the browser supplies only a relative path within the authorized workspace,
never a provider endpoint, sandbox ID or PVC name. This adds browsing, not
project editing or a general-purpose shell.

| Workspace state | How Files reads it |
| --- | --- |
| Sandbox running | Use the recorded, authorized attachment and OpenSandbox’s file API for directory listings and file bytes. No volume or agent run is required. Do not create a sandbox on a failed attach or silently switch to a different one. |
| Sandbox deleted; volume retained | Use a short-lived reader on the OpenSandbox side with only the recorded generation mounted read-only. It serves files without recreating the agent sandbox or provisioning its credentials. Release the mount after the read, including on cancellation. |
| No readable sandbox or retained volume | Report unavailable. Opening Files must never create an empty volume or present a provider failure as an empty folder. |

Both readers must confine lookup **and** opening the file to the workspace
root. The [native download handler][execd-read] accepts sandbox paths; prefixing
a user path with `/workspace` is not confinement. Require a trusted reader that
cannot follow links outside the root, including during concurrent changes. If
the live deployment cannot enforce this, do not expose raw SDK reads. Serve
only regular files, preserve the [existing listing filters](../sandbox.md#what-the-file-browser-shows-and-what-it-leaves-out),
and apply entry, byte and time bounds to listings, previews and downloads.

Live browsing can ship without Harness volume forwarding. Offline browsing
requires both durable storage **and** the read-only helper. If the helper
cannot mount that volume on the selected Docker host or Kubernetes cluster,
report offline browsing as unavailable; do not request a writable mount instead.

## Apply outbound policy to new compute

Under **Settings → Sandboxes → Connections**, let an operator choose server
defaults (omit policy), deny-all, a domain allowlist (default deny), or a domain
blocklist (default allow). Server defaults do not guarantee egress filtering.
Accept exact normalized domains only, not URLs, ports, IPs, CIDRs or wildcards.
Explain that [DNS and packet filtering][egress] admit resolved addresses for a
limited time; names sharing an address are not isolated from each other.
Deployment rules and port restrictions still apply.

Record the applied policy revision on the attachment. Policy changes govern
new work, not running processes. Refuse new work on a stale attachment,
including direct ref operations; never redirect an operation targeting that
ref to another sandbox. Retire the previous writer before replacing durable
compute. Require verified migration or explicit reset for legacy files. Keep
unversioned refs usable only under the unchanged legacy default.

## Grant destination credentials to private runs

The connection key authenticates AgenticOS to OpenSandbox. A **destination
credential** authenticates a matching outbound HTTPS request. Never place it
in the workspace, environment, command, spec, response, audit payload or logs.

- **Authorize the binding.** A principal with access to the specific organization
  [HTTP credential](../secrets.md) and authority to manage the agent binding
  approves the agent, run context, connection and endpoint revision. Validate at
  publish; before unsealing, recheck the caller, exposure, effective scope,
  destination, policy and endpoint. Changing the endpoint invalidates approval;
  never follow a redirect when delivering the secret.
- **Narrow the destination.** Require an explicit domain allow rule, the sealed
  credential’s [exact HTTPS origin](../../backend/app/core/secret_kinds.py) on
  port 443, allowed methods and narrow paths. Support bearer or named API-key
  headers; reject overlapping bindings, wildcards and personal credentials.
- **Isolate the run.** Use private run-scoped scratch only. No shared volumes or
  delegate handoff: guard publish and runtime, including nested delegates that
  would otherwise inherit the already-open backend.
- **Provision before handing out tools.** Commit an attempt with the approved
  endpoint and recovery identity. Create once with finite provider lifetime,
  default-deny policy and credential proxy; reconcile uncertain creation.
  Commit its sandbox ID **before** the vault write. Verify policy, install
  bindings via the trusted control plane and verify sanitized metadata before
  giving the agent access.
- **Recover and revoke.** Destroy on completion, failure or cancellation. A
  recovery worker claims abandoned attempts and retries on the original
  endpoint until absence is confirmed, without unsealing the secret. Revocation
  blocks new admissions; terminate the sandbox to end active use and revoke the
  upstream token in an emergency. Do not resume a credentialed run after its
  [sidecar vault][vault] has lost the binding.

## What to verify before enabling each feature

The [AgenticOS v1.93.0 lockfile][release-lock] pins Harness 0.71.0 and OpenSandbox
SDK 1.2.1. Outbound policy and credentials do **not** depend on the volumes PR:
Harness already forwards `network_policy` and `credential_proxy`. Wire them
into the adapter, leave `allow_internet_access` at its default with an explicit
policy, and replace `sandbox_timeout=None` with a finite lifetime for
credentialed runs only. Preserve existing lazy creation on unchanged connections.

| Feature | Acceptance check |
| --- | --- |
| Durable storage | On each supported runtime, write a file, destroy compute, mount the same volume in a different sandbox and compare bytes. Test cancelled creation and recovery, missing volumes, concurrent writers, tenant isolation and deletion without a live sandbox. |
| Migration | Interrupt before cutover: source intact. Complete cutover: destination matches. Delayed cleanup cannot delete the new generation. |
| Live Files | List, preview and download an agent-written file via the console; verify binary bytes. Deny unauthorized reads, path escapes and link swaps. A failed attach does not start compute. |
| Offline Files | Delete compute, then list and download the same bytes from the retained volume with no agent run or credentials. Enforce the same access checks and bounds; verify the mount is read-only, reader cleanup runs on cancellation, and missing storage is not recreated. |
| Outbound policy | Verify the target deployment’s egress image and `dns+nft` enforcement with an allowed name, denied name and disallowed address. A stale ref cannot start new work; a policy change does not discard files. |
| Credentials | Verify trusted TLS and authenticated vault delivery. With a throwaway token, only the approved origin, method and path receive it. Endpoint changes, revoked grants, cross-tenant use, shared scopes and delegate handoffs fail before tool use. Logs and metadata contain no token. |
| Cleanup | Stop the worker after a vault write; separately fail destruction and time out creation. Verify recovery on the original endpoint without duplicate creation, and provider expiry while cleanup is offline. |

Enable each feature per verified connection. Disabling a feature stops new use;
it must not erase retained files, lose recovery records or prevent cleanup.

[volumes-pr]: https://github.com/pydantic/pydantic-ai/pull/14282
[release-lock]: https://github.com/vstorm-co/agenticos/blob/v1.93.0/backend/uv.lock
[adapter]: https://github.com/vstorm-co/agenticos/blob/v1.93.0/backend/app/agents/capabilities/sandbox/_opensandbox.py
[docker-volume]: https://github.com/opensandbox-group/OpenSandbox/blob/release-1.2.1/docs/examples/docker-pvc-volume-mount.md
[k8s-volume]: https://github.com/opensandbox-group/OpenSandbox/blob/release-1.2.1/docs/examples/kubernetes-pvc-volume-mount.md
[lifecycle]: https://github.com/opensandbox-group/OpenSandbox/blob/release-1.2.1/specs/sandbox-lifecycle.yml
[execd-read]: https://github.com/opensandbox-group/OpenSandbox/blob/release-1.2.1/components/execd/pkg/web/controller/filesystem_download.go
[egress]: https://github.com/opensandbox-group/OpenSandbox/blob/release-1.2.1/docs/architecture/network/egress.md
[vault]: https://github.com/opensandbox-group/OpenSandbox/blob/release-1.2.1/docs/guides/credential-vault.md
