# OpenSandbox outbound domains and credentials

Status: proposed, 2026-11-20. SDK and repository review only; no live egress or
Credential Vault test was run.

## Recommendation

Ship outbound controls in two steps. First, give OpenSandbox connections an
operator-owned **Outbound access** setting: **Use server defaults**,
**Block outbound access**, **Allow only listed domains**, or **Block listed
domains**. Then let an
authorized operator grant a published agent use of specific **Outbound
credentials**. Start with run-scoped workspaces and exact HTTPS destinations.
The connection's API key still authenticates AgenticOS to the sandbox server;
it is never a credential for a destination the sandbox visits.

## What the pinned versions support

| Component | Finding |
| --- | --- |
| OpenSandbox Python SDK 1.2.1 | `Sandbox.create()` accepts `network_policy` and `credential_proxy=CredentialProxyConfig(enabled=True)`. Policy contains an ordered list of allow/deny `NetworkRule`s and a default action; credentials are not part of sandbox creation. After creation, `sandbox.credential_vault.create(credentials=..., bindings=...)` writes them to the egress sidecar for ordinary image-backed sandboxes. Reads return metadata, not values. [SDK source][sdk], [policy models][models], [Credential Vault guide][vault]. |
| Pydantic AI Harness 0.68.0 | The locked `OpenSandboxBackend` has `get_sandbox()` and a deny-all `allow_internet_access=False` option, but neither creation argument above nor a public creation-factory hook. Its `_create()` also owns the recovery tag and cancellation handling. Verified against the [published source archive][harness-068] and the [lockfile](../../backend/uv.lock). |
| Pydantic AI Harness 0.69.0 | The [published source archive][harness-069] accepts and forwards both arguments on `OpenSandboxBackend`. The higher-level `OpenSandbox` capability supports `network_policy` but not `credential_proxy`; the lower-level backend is the one AgenticOS already constructs. This change shipped in [Pydantic AI v2.69.0][pyd-269]; the harness requires `pydantic-ai-slim==2.69.0` rather than the locked 2.68.0. Review and test that coordinated upgrade before wiring the options. |

Use the new public constructor arguments, not a local rewrite of `_create()`.
Leave `allow_internet_access` at its default when supplying an explicit policy;
the harness rejects combining both controls. `get_sandbox()` gives the host an
SDK handle to initialize the vault before exposing the backend to agent tools.
Creation options do not retrofit a sandbox opened by reference. The SDK supports
reading the current policy and merging/deleting rules at runtime, but the patch
does not replace the whole policy or change its default action. Do not use it to
make an old workspace appear to match a new policy. [Harness API][harness-docs],
[SDK source][sdk].

## Where the settings belong

- **Settings → Sandboxes → Connections**: Show the outbound access setting only
  for OpenSandbox. Send no policy for **Use server defaults**. Use default-deny
  with no rules for **Block outbound access**, default-deny with allow rules for
  **Allow only listed domains**, and default-allow with deny rules for **Block
  listed domains**. Start with normalized, exact domain names (no schemes, ports,
  paths, IP/CIDRs or wildcards). A deny list is not an isolation policy; require
  the allowlist mode for credentialed sandboxes. Do not add credential hosts
  automatically or expose the vendor's policy JSON as a general agent setting.
- **Outbound credentials**: Create a scoped binding that an authorized operator
  can grant to a specific agent, and let the published agent reference that
  binding. Reuse the [HTTP credential](../secrets.md) vault kind
  ([`HttpCredentialSecret`](../../backend/app/core/secret_kinds.py)), which seals
  allowed origins with the token. Require the binding's exact HTTPS origin on
  port 443 to pass `allows()`, and require explicit methods, paths and auth type.
  Start with bearer and named API-key headers; avoid generic substitutions,
  exchange resolvers and personal account credentials in the first release. The
  destination must already be allowed by the effective connection policy. Do
  not let a spec pair arbitrary secret IDs with arbitrary URLs, substitute a
  generic API key, or inherit permission to use credentials from the default
  connection.
- **Enforce both authorizations**: Validate the reference at publish time and
  recheck it against the current run's principal/exposure, effective connection
  and grant before preparing the workspace. The existing
  [connection resolution](../../backend/app/services/sandbox_connection.py)
  checks organization and active state; the vault's
  [`resolve_for_bindings`](../../backend/app/services/organization_secret.py)
  is a lookup for an already-authorized binding, not a replacement for that
  check. Keep the vault value out of configuration, previews, command input and
  logs. Expose only metadata and safe failure categories. A matching destination
  can use the credential's authority and may reflect request data back; only
  grant narrowly scoped credentials to trusted upstreams.

## Creation, attach and cleanup

Keep the application-facing policy typed in the
[connection schema](../../backend/app/schemas/sandbox_connection.py), without
SDK field names or raw provider options. Resolve policy and credential authority
in the existing services. Keep SDK models, request translation and vault
provisioning inside the
[OpenSandbox adapter](../../backend/app/agents/capabilities/sandbox/_opensandbox.py),
behind the existing workspace interface. The
[workspace service](../../backend/app/services/sandbox_workspace.py) should
still own workspace references, concurrency control and cleanup. Put preparation
on this common path rather than in tools so no file or command operation can
skip it. Keep lazy creation for unchanged connections, and preserve no-retry
creation, safe error handling and attach-by-reference behavior.

For a new credentialed workspace, create with default-deny policy and proxy
enabled, record its sandbox ID for cleanup, then populate the vault control-plane
to control-plane. Verify the applied policy and sanitized binding metadata before
making the workspace available. A failed or ambiguous provisioning request must
not allow a run to continue without its credential setup. Reconcile or destroy
that sandbox through the existing durable cleanup path; do not blindly retry
creation. Recheck grants for future runs; for emergency revocation, revoke the
upstream credential as well, since a sandbox can already hold an injected grant.

For egress-only persistent workspaces, record which policy revision was applied
to the ref. When settings differ, refuse new work on the existing sandbox with an
explicit reset/move message; do not silently keep the old policy or replace the
workspace and lose access to its files. A never-configured policy keeps legacy
refs working; a newly configured policy must not treat a legacy ref as verified.
Treat policy changes as effective on new workspaces; coordinate explicit
termination if old running work must stop immediately. Credentialed workspaces
stay run-scoped initially, because conversation, channel, agent, user and project
[scopes](../../backend/app/agents/capabilities/sandbox/_identity.py) can outlive a
run or share a ref across distinct authorities. For classic sidecar sandboxes,
Kubernetes pause/resume loses the in-memory vault; Docker pause/unpause preserves
it unless the sidecar restarts or is replaced. Do not promise persistent injection
or automatically repopulate a shared workspace on attach. [Vault lifecycle][vault].

This is for OpenSandbox-backed agent workspaces. Do not advertise it for
[workflow code sandboxes](../../backend/app/workflows/nodes/_sandbox_job.py),
which use sandboxd, or for other providers. Do not claim that it governs browser
or backend HTTP tools that do not execute in the sandbox.

## Deployment gate

- This is a separate OpenSandbox server deployment, not enabled by installing the
  Python SDK. Pin a compatible server, controller/execd where applicable and
  egress image, and configure `[egress].image` and `[egress].mode = "dns+nft"`.
  The default `dns` only filters DNS and cannot enforce an outbound allowlist at
  the packet layer; the vault refuses DNS-only mode. Operator rules and TCP port
  restrictions can override policy expectations. [Server configuration][server],
  [network security guide][network].
- Validate the supported runtime explicitly. Docker requires bridge networking
  for policy. The ordinary Kubernetes sidecar needs `NET_ADMIN` and supported
  nftables; gVisor and transparent service-mesh injection are not compatible
  with this path. For credentials, check the workload's TLS client trusts the
  provisioned interception CA, upstream TLS is verified, and vault writes use
  an authenticated, protected control-plane path. Different routing profiles
  (including template-backed Fast Sandbox) are outside the first rollout.
  [Egress architecture][egress], [Credential Vault guide][vault].
- **Secure Access** is inbound endpoint authentication, not outbound filtering.
  The [recorded OpenShift validation](../kubernetes.md#oidc-and-sandbox-configuration)
  exercised Secure Access and reattachment, not egress, interception or the
  sidecar's SCC/capabilities. Do not treat it as a pass for this feature.
- Extend the disposable connection probe or operator acceptance check to report
  shell reachability separately from egress/credential support. On the intended
  deployment, verify an allowed test domain works, a disallowed one and
  unauthorized direct-IP traffic fail, and approved requests to a controlled
  HTTPS endpoint receive only a throwaway credential. Also verify unapproved
  methods/paths get no credential, policy/grant mismatch never reaches tools,
  and provisioning failure destroys or queues cleanup of the sandbox. Do not put
  real provider credentials in a probe. Until tested, mark the deployment
  unverified instead of inferring support from the server version or shell probe.

[sdk]: https://github.com/opensandbox-group/OpenSandbox/blob/release-1.2.1/sdks/sandbox/python/src/opensandbox/sandbox.py
[models]: https://github.com/opensandbox-group/OpenSandbox/blob/release-1.2.1/sdks/sandbox/python/src/opensandbox/models/sandboxes.py
[vault]: https://github.com/opensandbox-group/OpenSandbox/blob/release-1.2.1/docs/guides/credential-vault.md
[harness-068]: https://pypi.org/project/pydantic-ai-harness/0.68.0/#files
[harness-069]: https://pypi.org/project/pydantic-ai-harness/0.69.0/#files
[pyd-269]: https://github.com/pydantic/pydantic-ai/releases/tag/v2.69.0
[harness-docs]: https://pydantic.dev/docs/ai/harness/opensandbox-sandbox/#keep-a-token-out-of-the-sandbox
[server]: https://github.com/opensandbox-group/OpenSandbox/blob/release-1.2.1/server/configuration.md#egress
[network]: https://github.com/opensandbox-group/OpenSandbox/blob/release-1.2.1/docs/guides/network-security.md
[egress]: https://github.com/opensandbox-group/OpenSandbox/blob/release-1.2.1/docs/architecture/network/egress.md
