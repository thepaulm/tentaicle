# Tentaicle implementation plan

## Summary

Build a personal remote-terminal application for Linux and macOS. Users authenticate with Google, enroll machines, list available machines and sessions, and connect through encrypted WebRTC data channels.

Use Go and [Pion WebRTC](https://github.com/pion/webrtc). A trusted coordinator provides identity, machine discovery, authorization, and signaling. Terminal traffic travels directly between peers when possible, with the existing AWS TURN server providing relay fallback.

Sessions run under the agent’s OS account and survive network disconnects. They do not survive agent restarts or machine reboots. No tmux dependency.

This document records the agreed plan. Application implementation has not started.

## Architecture and interfaces

### Executables and commands

Maintain one Go module producing a `tentaicle` CLI/agent binary and a `tentaicle-server` coordinator binary. Keep authentication, signaling, transport, and terminal management in separate internal packages.

Provide these commands:

| Command | Behavior |
|---|---|
| `tentaicle login` | Print a browser approval URL and code; save the resulting device credential |
| `tentaicle logout` | Revoke this CLI credential and remove it locally |
| `tentaicle machines` | List owned machines, availability, and last-seen time |
| `tentaicle connect [machine]` | Select a machine and existing session, or create a session |
| `tentaicle sessions <machine>` | List that machine’s sessions |
| `tentaicle attach <machine> <session>` | Attach directly to an existing session |
| `tentaicle terminate <machine> <session>` | Explicitly terminate a remote session |
| `tentaicle agent enroll --name <name>` | Approve enrollment through a browser and register this OS account |
| `tentaicle agent run` | Run the foreground agent |
| `tentaicle agent install/uninstall` | Install or remove the user service |
| `tentaicle machine revoke <machine>` | Disable the machine’s credential and remote access |
| `tentaicle devices` / `device revoke <id>` | List and revoke CLI credentials |
| `tentaicle doctor` | Diagnose coordinator, authentication, and ICE connectivity |

Use interactive selection when arguments are omitted in a terminal. Require explicit arguments in noninteractive use. Provide JSON output for listing commands.

Configuration precedence: command flags, `TENTAICLE_*` environment variables, then a per-user configuration file. Keep credentials separate with owner-only permissions; never place them in URLs, command arguments, or logs.

### Coordinator

Expose versioned HTTPS JSON APIs under `/v1` and authenticated WebSocket signaling.

The API supports browser approval requests, credential refresh/revocation, enrollment, machine discovery, and connection requests. WebSocket messages carry request IDs and explicit protocol versions, covering presence, session summaries, authorization grants, SDP, and ICE candidates.

Use SQLite with migrations and WAL mode on persistent storage. Store users, enrolled agents, CLI devices, hashed opaque credentials, approval requests, and access-event metadata. Identify Google users by issuer and `sub`, not email.

Agents maintain outbound WebSocket connections. Heartbeats run every 15 seconds; mark a machine offline after 45 seconds without contact. Publish session summaries without terminal contents. Reconnecting agents replace stale presence and session inventories.

The coordinator is a single instance for v1. Coordinator restarts preserve enrollment and credentials; agents reconnect with capped exponential backoff and jitter.

### Authentication and authorization

Implement an application-owned browser approval flow:

1. CLI or agent requests a short-lived approval transaction and receives a secret polling token plus a human-readable code.
2. The user opens the coordinator’s approval page on any browser, signs in with Google, and explicitly approves the displayed client or machine.
3. The requesting process polls using its secret and receives a dedicated revocable credential.

Use Google’s server-side authorization-code flow with exact registered HTTPS callbacks, state, nonce, secure browser sessions, and validated ID tokens. Request only identity scopes. This approval mechanism is our own; do not assume Google supports device authorization for this use case.

Approval codes expire after 10 minutes and are single-use. Rate-limit creation, code entry, and polling. Browser approval must require an explicit action protected against CSRF.

Issue short-lived access tokens and rotating opaque refresh credentials. Store refresh credentials hashed on the server. Reuse of a rotated refresh credential revokes that device’s credential family.

Each agent belongs to one user and one local OS account. Agents can publish their own state and accept authorized connections; they cannot enumerate or control other machines.

For every connection, issue a signed, single-use grant bound to the user, CLI device, target agent, connection ID, requested action, and expiry. The agent verifies it before exposing terminal operations.

Trust the coordinator as the authorization authority. WebRTC encryption and authenticated signaling protect connections; independent peer-key pinning is outside v1.

Push revocation to connected agents immediately. Renew active connection authorization every 30 seconds with a 90-second lease. If renewal fails, detach remote clients at lease expiry while retaining local shells. Revoking an enrolled agent also prevents further enrollment-credential use.

### Transport

Use Pion’s reliable, ordered WebRTC data channels with ICE and authenticated coordinator signaling.

Define one versioned application channel carrying bounded frames for:

- Session creation and attachment.
- Terminal input and sequenced output.
- Terminal resize.
- Detach, termination, exit status, errors, and heartbeat.

Keep terminal bytes as bytes; do not reinterpret their encoding. Chunk payloads to at most 16 KiB and bound queues. Isolate WebRTC backpressure from PTY reading so slow clients cannot exhaust agent memory.

Attempt ICE connection establishment for 30 seconds, then return an actionable failure. Include an explicit relay-only diagnostic option and report whether the chosen path is direct or relayed.

Support configurable STUN/TURN URLs and either static TURN credentials or shared-secret-generated expiring credentials. Require the operator to select the credential mode. Verify the existing server’s supported transports and authentication before deployment acceptance.

## Terminal and platform behavior

Create each session with a PTY, the account’s configured login shell, and its home directory as the starting directory. No agent-managed account switching or root execution. Users may invoke ordinary `sudo` inside the shell.

Support multiple sessions per machine, with a default limit of eight. Allow one attached client per session; reject additional attachments with a clear “already attached” response.

Place the local terminal into raw mode while attached and always restore its state on exit or failure. Propagate window sizes and resize events. Deliver Ctrl-C through the PTY. Reserve `Ctrl-]` followed by `d` for detach, with an escaping mechanism for literal input.

Keep disconnected shells running until explicit termination, shell exit, or agent shutdown. Disable automatic idle expiry by default.

Maintain a 1 MiB in-memory output ring per session with monotonic byte offsets:

- On transport loss, attempt reconnection for up to 60 seconds.
- A reconnecting client requests output after its last received offset.
- A new attachment receives the retained output.
- Report truncation visibly when older output has been discarded.
- Never resend uncertain terminal input after reconnecting; tell the user input may need to be re-entered.
- Full-screen applications may require a redraw; exact screen reconstruction is outside v1.

Session termination sends a graceful termination signal, then force-kills remaining session processes after five seconds. Agent shutdown terminates its sessions and reaps children. Retain exited-session metadata and buffered output for ten minutes so reconnecting clients can observe exit status.

Install user services using systemd on Linux and a LaunchAgent on macOS. Document Linux lingering as an optional operator setup for operation without login. macOS support initially assumes the user is logged in; logout survival is not promised.

Build and test Linux and macOS binaries for amd64 and arm64. Provide release archives, checksums, and installation instructions.

## Deployment, operations, and milestones

### AWS deployment

Use Terraform to provision one EC2 instance, encrypted persistent EBS storage, security groups, an instance role, and scheduled EBS snapshots.

Defaults:

- AWS region `us-west-2`, configurable.
- Ubuntu LTS on a small amd64 instance.
- Existing Route 53 hosted zone and a supplied coordinator hostname.
- Public HTTPS on port 443; port 80 only for certificate issuance and redirect.
- Administration through AWS Systems Manager; no public SSH.
- Caddy for TLS and reverse proxying.
- Coordinator container pinned to a release image digest.
- SQLite stored on a separately mounted data volume retained on instance termination.

Terraform accepts references to runtime secrets; do not put Google or TURN secrets in Terraform variables or state. Fetch secrets through the instance role from AWS Secrets Manager.

Create daily SQLite online backups and retain seven daily copies on the data volume. Snapshot the volume daily with seven-day retention. Document and test restoration onto a replacement host.

Expose liveness and readiness endpoints. Emit structured logs and counters for login failures, online agents, connection attempts, relay selection, disconnects, and authorization failures. Record access events for 30 days. Never record terminal contents or credentials.

Document initial Google OAuth setup, allowed Google user IDs, DNS, secret provisioning, TURN compatibility checks, upgrades, backup restoration, and device revocation. Restrict the deployment to explicitly allowed users.

### Implementation sequence

1. **Transport proof:** Establish Go-to-Go data channels with local signaling; verify direct and forced TURN connections using the existing server.
2. **Identity and coordination:** Implement Google approval, device credentials, enrollment, discovery, signaling, and authorization grants.
3. **Interactive terminal:** Add PTYs, session selection, input/output, resize, detach, and exit handling.
4. **Recovery and access control:** Add bounded replay, reconnect, backpressure, revocation leases, and process cleanup.
5. **Distribution and AWS:** Add platform services, release builds, Terraform, HTTPS, backups, diagnostics, and operator documentation.

Each milestone includes its relevant automated tests. Complete the TURN proof before building features that depend on deployment connectivity.

## Validation and acceptance

Use unit tests for authorization, credential rotation, approval expiry, protocol bounds, replay offsets, and session ownership. Use integration tests with a fake identity provider, a real coordinator, two Pion peers, and a local TURN server.

Acceptance requires:

- Headless CLI login and agent enrollment approved in another device’s browser.
- Owned machines listed correctly; other users cannot discover or access them.
- Interactive shells work on Linux and macOS, including Unicode, Ctrl-C, resizing, and terminal-state restoration.
- Direct and forced-relay connections both pass against the deployment’s TURN server.
- Disconnect and reattach preserve a running shell without duplicating input.
- Output overflow, slow clients, and continuous detached output keep memory bounded.
- Concurrent attachment is rejected consistently.
- Expired, replayed, revoked, wrong-agent, and wrong-user grants are rejected.
- Credential revocation disconnects active clients within 90 seconds even if a push is lost.
- Coordinator restart preserves enrollment; agent restart clearly invalidates previous sessions.
- Backup restoration recovers users, enrollment, and credentials.
- A clean AWS deployment can be reproduced from Terraform and documented setup inputs.

Defer file transfer, remote-command automation, team sharing, Windows, browser terminals, independent peer authentication, exact terminal screen restoration, and high availability until after this release.
