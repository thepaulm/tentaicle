# Tentaicle implementation plan

## Summary

Build a personal remote-terminal application for Linux and macOS. Users authenticate with Google, enroll machines, list available machines and sessions, and connect through encrypted WebRTC data channels.

Use Go and [Pion WebRTC](https://github.com/pion/webrtc). A trusted coordinator provides identity, machine discovery, authorization, and signaling. Any explicitly allowed user who completes Google sign-in may reach their enrolled machines; no per-device pairing or access to the remote host is required. Terminal traffic travels directly between peers when possible, with the existing AWS TURN server providing relay fallback.

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
| `tentaicle attach <machine> <session> [--takeover]` | Attach directly to an existing session, optionally displacing the attached client |
| `tentaicle terminate <machine> <session>` | Explicitly terminate a remote session |
| `tentaicle local new [--name <name>]` | Create a session on this machine’s agent and attach to it |
| `tentaicle local sessions` | List this machine’s agent sessions |
| `tentaicle local attach <session> [--takeover]` | Attach to a session on this machine’s agent |
| `tentaicle agent enroll --name <name>` | Approve enrollment through a browser and register this OS account |
| `tentaicle agent run` | Run the foreground agent |
| `tentaicle agent start [--inhibit-sleep]` / `stop` / `status` | Start the agent in the background, stop it, or report its state |
| `tentaicle machine revoke <machine>` | Disable the machine’s credential and remote access |
| `tentaicle devices` / `device revoke <id>` | List and revoke CLI credentials |
| `tentaicle doctor` | Diagnose coordinator, authentication, and ICE connectivity |

Use interactive selection when arguments are omitted in a terminal. Require explicit arguments in noninteractive use. Provide JSON output for listing commands.

Configuration precedence: command flags, `TENTAICLE_*` environment variables, then a per-user configuration file. Keep credentials separate with owner-only permissions; never place them in URLs, command arguments, or logs.

### Coordinator

Expose versioned HTTPS JSON APIs under `/v1` and authenticated WebSocket signaling.

The API supports browser approval requests, credential refresh/revocation, enrollment, machine discovery, and connection requests. WebSocket messages carry request IDs and explicit protocol versions, covering presence, session summaries, authorization grants, SDP, and ICE candidates.

Use SQLite with migrations and WAL mode on persistent storage. Store users, enrolled agents and their public keys, CLI devices, hashed opaque credentials, approval requests, and access-event metadata. Identify Google users by issuer and `sub`, not email.

Agents maintain outbound WebSocket connections. Heartbeats run every 15 seconds; mark a machine offline after 45 seconds without contact. Publish session summaries with ID, name, creation time, and attachment state (detached, attached locally, or attached remotely), without terminal contents. Reconnecting agents replace stale presence and session inventories.

The coordinator is a single instance for v1. Coordinator restarts preserve enrollment and credentials; agents reconnect with capped exponential backoff and jitter.

### Authentication and authorization

Implement an application-owned browser approval flow:

1. CLI or agent requests a short-lived approval transaction and receives a secret polling token plus a human-readable code.
2. The user opens the coordinator’s approval page on any browser, signs in with Google, and explicitly approves the displayed client or machine.
3. The requesting process polls using its secret. A CLI receives a dedicated revocable credential; an agent’s public key, submitted with its approval request, becomes registered.

Use Google’s server-side authorization-code flow with exact registered HTTPS callbacks, state, nonce, secure browser sessions, and validated ID tokens. Request only identity scopes. This approval mechanism is our own; do not assume Google supports device authorization for this use case.

Approval codes expire after 10 minutes and are single-use. Rate-limit creation, code entry, and polling. Browser approval must require an explicit action protected against CSRF.

CLI devices receive short-lived access tokens and rotating opaque refresh credentials. Store refresh credentials hashed on the server. Reuse of a rotated refresh credential revokes that device’s credential family, except that the immediately previous credential is accepted once within 60 seconds of rotation if its replacement has not been used, so a lost refresh response does not revoke the device.

Agents do not use refresh credentials. At enrollment, the agent generates an Ed25519 key pair, keeps the private key in its state directory with owner-only permissions, and registers the public key. The agent authenticates each WebSocket connection and access-token request by signing a coordinator-issued nonce. Agent credentials do not expire from inactivity; they end only by revocation. This keeps an unattended agent from stranding itself through a failed rotation.

Each agent belongs to one user and one local OS account. Agents can publish their own state and accept authorized connections; they cannot enumerate or control other machines.

For every connection, issue a signed, single-use grant bound to the user, CLI device, target agent, connection ID, the CLI’s DTLS certificate fingerprint, requested action, and expiry. The agent verifies it, allowing 60 seconds of clock skew, and checks that the fingerprint matches the connected peer’s DTLS certificate before exposing terminal operations. A grant therefore cannot be reused on another connection.

Trust the coordinator as the authorization authority: access depends on Google sign-in by an allowed user and a valid coordinator credential. WebRTC encryption and authenticated signaling protect connections. A compromised coordinator could grant itself access; independent peer-key pinning is outside v1.

Push revocation to connected agents immediately. Renew active connection authorization every 30 seconds with a 90-second lease. If renewal fails, detach remote clients at lease expiry while retaining local shells. Revoking an enrolled agent disables its registered key.

### Transport

Use Pion’s reliable, ordered WebRTC data channels with ICE and authenticated coordinator signaling. Mandatory DTLS encrypts every channel end to end, including through TURN; add no second encryption layer. Use only pure-Go cryptography (no OpenSSL or cgo).

The CLI sends its grant as the first frame; the agent accepts no other frames until the grant verifies.

Define one versioned application channel carrying bounded frames, each tagged with a stream type so later features (such as file transfer) need no protocol version change. Frames cover:

- Session creation and attachment.
- Terminal input and sequenced output.
- Terminal resize.
- Detach, termination, exit status, errors, and heartbeat.

Keep terminal bytes as bytes; do not reinterpret their encoding. Chunk payloads to at most 16 KiB and bound queues. Isolate WebRTC backpressure from PTY reading so slow clients cannot exhaust agent memory.

Attempt ICE connection establishment for 30 seconds, then return an actionable failure. Include an explicit relay-only diagnostic option and report whether the chosen path is direct or relayed.

Support configurable STUN/TURN URLs and either static TURN credentials or shared-secret-generated expiring credentials. Require the operator to select the credential mode. Verify the existing server’s supported transports and authentication before deployment acceptance.

## Terminal and platform behavior

Create each session with a PTY, the account’s configured login shell, and its home directory as the starting directory.

Start sessions from a minimal environment: `HOME`, `USER`, `LOGNAME`, `SHELL`, a default `PATH`, and `TERM` and locale variables supplied by the creating client; the login shell’s profile provides the rest. Do not inherit the agent’s environment, so values such as `SSH_AUTH_SOCK`, `DISPLAY`, and `DBUS_SESSION_BUS_ADDRESS` from wherever the agent was started do not leak into sessions. Set `TENTAICLE_SESSION` to the session ID so the CLI warns before nesting sessions. Document that a session keeps the `TERM` it was created with; attaching from a different terminal type may render imperfectly. No agent-managed account switching or root execution. Users may invoke ordinary `sudo` inside the shell.

Support multiple sessions per machine, with a default limit of eight. Sessions have an ID and an optional user-supplied name; commands accept either. Allow one attached client per session, local or remote; reject additional attachments with a clear “already attached” response that offers takeover. With `--takeover` (or confirmation in interactive `connect`, which shows each session’s attachment state), an authorized client of the same user displaces the attached client, which is told it was taken over and detached. Takeover lets a user recover a session held by a dead client without waiting for its reconnect window, or one left attached in a local terminal.

Place the local terminal into raw mode while attached and always restore its state on exit or failure. Propagate window sizes and resize events. Deliver Ctrl-C through the PTY. Reserve `Ctrl-]` followed by `d` for detach and `r` for redraw, with an escaping mechanism for literal input.

Keep disconnected shells running until explicit termination, shell exit, or agent shutdown. Disable automatic idle expiry by default.

Maintain a 1 MiB in-memory output ring per session with monotonic byte offsets:

- On transport loss, attempt reconnection for up to 60 seconds. The agent releases a remote attachment after the same window.
- A reconnecting client requests output after its last received offset.
- A new attachment receives the retained output.
- Report truncation visibly when older output has been discarded.
- Input frames carry per-attachment sequence numbers; the agent acknowledges the last sequence written to the PTY. On reconnect, the agent reports its last applied sequence and the client resends only later frames, so input is neither duplicated nor lost. Bound unacknowledged input to 64 KiB and stop reading local input, other than the detach escape, while reconnecting.
- When replaying retained output, the client strips sequences that make the local terminal reply or act, such as status and cursor-position reports, device-attribute queries, color and setting queries, window reports, and OSC 52 clipboard writes. The filter handles sequences split across chunks. Live output is not filtered.
- After every attachment, and on redraw, the agent briefly changes the PTY size and restores it, because the kernel signals only actual size changes. This prompts full-screen applications to redraw. Exact screen reconstruction is outside v1.

Session termination sends a graceful termination signal, then force-kills remaining session processes after five seconds. Agent shutdown terminates its sessions and reaps children. Retain exited-session metadata and buffered output for ten minutes so reconnecting clients can observe exit status.

Run the agent as a background process, as tmux does; install no systemd units, LaunchAgents, or system services. The user logs in once and runs `tentaicle agent start`, which re-executes the binary in a new session (`setsid`) with no controlling terminal and standard streams redirected, so it survives SSH disconnect and logout. Go cannot safely `fork` without `exec`, so use re-execution rather than a classic double fork.

- Hold an exclusive lock and PID file in the user’s state directory so only one agent runs per account; `agent stop` and `agent status` use it.
- Write logs to a size-bounded, rotated file in the state directory with owner-only permissions.
- The agent does not restart after a crash and does not start at boot; the user logs in and runs `agent start` again. Previous sessions are gone, as with any agent restart.
- On Linux, logout survival requires logind’s `KillUserProcesses=no`, the default on most distributions. `agent start` and `tentaicle doctor` read the setting from logind and warn when it is enabled.
- On macOS, the agent survives SSH disconnect; survival of GUI logout is not promised.

Build and test Linux and macOS binaries for amd64 and arm64. Provide release archives, checksums, and installation instructions.

### Local attachment

Users can create and attach to agent sessions on the machine itself, as with tmux, so work started locally can be resumed remotely and vice versa. A shell in an ordinary terminal window is not an agent session and cannot be attached remotely.

- The agent listens on a Unix socket in its state directory (`~/.local/state/tentaicle` on Linux, `~/Library/Application Support/tentaicle` on macOS) with owner-only permissions, and recreates the socket if it is removed. Do not use `$XDG_RUNTIME_DIR` or `/tmp`: logind deletes the runtime directory when the user’s last session ends, and temporary-file cleaners remove old sockets. It verifies the connecting process’s UID with `SO_PEERCRED` on Linux and `LOCAL_PEERCRED` on macOS, and rejects other users.
- Local clients use the same terminal frames as remote clients, without grants, leases, or the coordinator; local attachment works while the coordinator is unreachable.
- Local and remote attachments share the session limit, the single-attachment rule, takeover, replay, and replay filtering.
- Closing the local terminal detaches; the session keeps running.

### Sleep inhibition

With `agent start --inhibit-sleep` or the equivalent configuration setting, the agent prevents idle system sleep while it runs. It is off by default.

- On Linux, hold a logind `sleep` inhibitor lock in block mode over D-Bus, using a pure-Go client. Do not inhibit `idle`: some desktops treat that as user activity and never lock the screen.
- On macOS, run `caffeinate -i -w <agent pid>` as a child process, since the native power-assertion API requires cgo. This permits display sleep and screen locking.
- Inhibition does not override lid close, forced suspend, or administrator power policy.
- `agent status` and `tentaicle doctor` report whether inhibition is active, and warn when it was requested but could not be acquired.

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
3. **Interactive terminal:** Add PTYs, session selection and naming, local attachment, input/output, resize, detach, and exit handling.
4. **Recovery and access control:** Add bounded replay with query filtering, reconnect with input sequencing, takeover, backpressure, revocation leases, and process cleanup.
5. **Distribution and AWS:** Add background agent mode, sleep inhibition, release builds, Terraform, HTTPS, backups, diagnostics, and operator documentation.

Each milestone includes its relevant automated tests. Complete the TURN proof before building features that depend on deployment connectivity.

## Validation and acceptance

Use unit tests for authorization, credential rotation, approval expiry, protocol bounds, replay offsets, replay filtering, input sequencing, DTLS fingerprint binding, local socket peer-UID checks, agent key authentication, the refresh grace window, session environment construction, and session ownership. Use integration tests with a fake identity provider, a real coordinator, two Pion peers, and a local TURN server.

Acceptance requires:

- Headless CLI login and agent enrollment approved in another device’s browser.
- Owned machines listed correctly; other users cannot discover or access them.
- Interactive shells work on Linux and macOS, including Unicode, Ctrl-C, resizing, and terminal-state restoration.
- Direct and forced-relay connections both pass against the deployment’s TURN server.
- Disconnect and reattach preserve a running shell without duplicating or losing input.
- Replayed output containing terminal queries produces no input to the remote shell.
- Output overflow, slow clients, and continuous detached output keep memory bounded.
- Concurrent attachment is rejected consistently; takeover detaches the previous client.
- A session created locally can be detached, attached remotely, and attached locally again, with takeover working in both directions.
- Another local user cannot connect to the agent’s socket.
- With sleep inhibition enabled, the machine does not idle-sleep while the agent runs on Linux and macOS, the screen still locks on its idle timer, and inhibition ends when the agent stops.
- A lost refresh response does not revoke a CLI device; an agent stays enrolled across coordinator and network outages of any length.
- Session listings show attachment state; full-screen applications redraw after attaching.
- Sessions do not inherit the agent’s environment.
- Expired, replayed, revoked, wrong-agent, and wrong-user grants are rejected.
- A grant presented on a connection whose DTLS fingerprint does not match is rejected.
- A Google account not on the allow-list cannot log in, enroll, or connect.
- Credential revocation disconnects active clients within 90 seconds even if a push is lost.
- The agent survives SSH disconnect and logout on Linux with `KillUserProcesses=no`, and on macOS after SSH disconnect.
- Coordinator restart preserves enrollment; agent restart clearly invalidates previous sessions.
- Backup restoration recovers users, enrollment, and credentials.
- A clean AWS deployment can be reproduced from Terraform and documented setup inputs.

Defer file transfer, remote-command automation, team sharing, Windows, browser terminals, independent peer authentication, service-manager integration (systemd user units with lingering, LaunchAgents) for crash restart and boot start, support for networks that allow only web ports (TURN over TLS on 443), exact terminal screen restoration, and high availability until after this release.
