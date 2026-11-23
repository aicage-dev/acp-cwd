# ACP CWD Proxy — Project Specification

**Status:** Design proposal / implementation handoff  
**Created:** 2026-11-23  
**Working name:** `acp-cwd` (not final)

## 1. Problem

Some ACP clients start an agent executable with a process working directory unrelated to the opened project. JetBrains Air, for example, has been observed to launch custom agents from an internal directory such as:

```text
~/.cache/JetBrains/IntelliJIdea2026.3/air/acp-process/acp_opencode_aicage_logged
```

JetBrains AI Assistant, by contrast, has been observed launching the same kind of external agent from the actual project directory.

ACP supplies the project directory separately, in session-related messages (e.g. `session/new`, `session/list`). This is valid protocol behavior: an agent must use the session's `cwd` even when it differs from the process launch directory.

However, some agent launchers and tools **must know the project directory before starting**. In particular, `aicage` selects project-specific configuration and Docker mounts based on its process CWD. If it is launched from an IDE-internal directory, it may select the wrong configuration and fail to mount the actual project. OpenCode inside aicage then fails `session/new` with `LocationNotFoundError` because the requested project directory is unavailable in the container.

### Observed behavior (2026-11-23)

| Client / launch | Initial process CWD | Outcome |
|---|---|---|
| JetBrains AI Assistant → `aicage opencode acp` | Actual project | Works |
| JetBrains Air → `aicage opencode acp` | Air-internal ACP directory | `session/new` → `LocationNotFoundError` without correct mount |
| JetBrains Air → `codex-acp` with `CODEX_PATH=aicage-codex` | Air-internal ACP directory | Works via Codex ACP adapter |

The Codex case uses `codex-acp`, a bridge to the Codex App Server protocol; it is **not** evidence that direct ACP-to-ACP proxying already solves this problem. In the working setup, `codex-acp` launches `aicage-codex` using `CODEX_PATH`.

A temporary workaround was confirmed: create aicage configuration for Air's internal directory and mount the real project there. This is unsuitable across many projects and IDE-generated directories.

## 2. Goal

Build a **standalone, generic ACP stdio proxy** that lets an ACP client launch any configured agent from an arbitrary process CWD, while ensuring the actual agent process used for a project starts **with that project's CWD**.

Intended invocation:

```sh
acp-cwd -- aicage opencode acp
```

The proxy should be usable by aicage **and other launchers/agents**, without per-project configuration and without requiring IDE/plugin authors to provide launch-time project paths.

### Non-goals

- Modify aicage's project selection rules, silently mount HOME, or add IDE-specific heuristics.
- Require every IDE or ACP client to set environment variables, use placeholders, or supply the project as a CLI argument.
- Build a Codex-specific protocol bridge.
- Claim universal transparency for arbitrary undocumented agent extensions without testing them.
- Make the initial process CWD equal to the project CWD before ACP reveals the project; that is impossible without another information source.

## 3. Protocol constraint

A typical ACP v1 exchange begins:

```text
Client                         Proxy                        Agent A (probe)
  | -- initialize ----------->   | -- initialize ----------->  |
  | <---------- result --------  | <---------- result --------  |
  | -- pre-session requests ->   | <------ forward --------->  |
  | -- session/list or new -->   |  <-- first project cwd      |
```

`initialize` normally contains version, client capabilities, client identity, and extension fields, **not the project CWD**. The response contains the negotiated protocol version, agent capabilities, authentication methods, agent identity, and optional extension fields.

The project path can appear in `session/list`, `session/new`, `session/load`, `session/resume`, or other protocol messages. Do not hardcode the assumption that `session/new` is the first message with a project directory. Extract it from **the earliest legitimate, well-defined project-CWD field** observed, while preserving protocol semantics. Avoid treating arbitrary strings called `cwd` deep inside opaque extension payloads as authoritative without a deliberate policy.

A client may wait for a real `initialize` response before sending any message that contains project CWD. Consequently the proxy cannot simply defer **all** agent startup until a project path arrives.

## 4. Proposed architecture: probe, then project agent

**Key idea:** Start the real agent once as a *probe* to get genuine initialization behavior, then switch to a project-specific real agent when ACP reveals the project directory. The proxy owns the permanent client-facing connection.

1. Client launches `acp-cwd` from any directory, passing the intended agent executable and arguments.
2. Proxy starts **Agent A** with the initial inherited CWD and environment.
3. Proxy forwards the client's complete `initialize` request to Agent A and forwards the real response to the client. Preserve unknown fields and extensions.
4. Until a usable project CWD appears, proxy handles pre-project messages via Agent A, while tracking the consequences for a later switch.
5. On the first valid project CWD, proxy starts **Agent B** with that directory as its **process CWD**, otherwise preserving the configured executable, arguments, and environment.
6. Proxy sends Agent B a complete initialization request equivalent to the original one. The internal response is **not sent to the client**. The proxy verifies compatibility with what was already advertised.
7. Proxy routes the triggering request and following traffic to Agent B, with necessary request/response correlation and any explicitly implemented state translation.
8. Agent A can be stopped once it has no outstanding obligations; do not blindly kill it while requests, authentication, or notifications still depend on it.
9. Client-facing stdin/stdout and connection identity remain stable across the internal handoff.

**Optimization:** If the inherited launch CWD is already the requested project CWD, Agent A may be retained instead of starting Agent B. This is an optimization, not a prerequisite for the first implementation.

### Important distinction

The initial agent and project agent have **different internal ACP connections**. This is not a process restart that leaks a second `initialize` response to the client. The proxy mediates two connections and owns their state. Still, that does not automatically make pre-project requests/statements transferable: see below.

## 5. Initialization data that must survive the handoff

### Store the complete client `initialize` request

Do **not** reconstruct only known fields. Keep the full original JSON-RPC message and its `params`, including:

- `protocolVersion` and negotiated version behavior.
- `clientCapabilities` and `clientInfo` (v1), plus all supported v2 variations.
- `_meta`, custom extensions, and any future fields.
- JSON-RPC request ID for matching the **client-facing** response; internally generated request IDs can differ.

The second agent's initialization must receive semantically identical initialization parameters. Preserve unknown fields where supported; do not silently rewrite identity or capabilities.

### Store the first agent's complete `initialize` response

Use this as the client-visible contract:

- Negotiated `protocolVersion`.
- `agentCapabilities` (v1) / equivalent v2 capability structures.
- `authMethods` and related extensions.
- `agentInfo` and arbitrary `_meta`/extension fields.

When initializing Agent B, compare the new response against this contract. The practical comparison policy needs careful design: matching a protocol version and honoring every capability **already advertised** is essential; byte-for-byte equality of informational fields may be unnecessary. On an incompatibility, fail clearly rather than silently changing the capabilities the client believes it has.

**Standard initialization does not create an ACP session ID.** Session identifiers arrive later. Agent-specific initialization extensions might carry stateful identifiers or connection-scoped tokens; these require explicit handling or a guarded compatibility failure.

## 6. Pre-project traffic is the principal risk

Before `cwd` appears, the client may send requests besides `initialize`. Observed examples:

- `providers/list` (Air with `codex-acp`).
- `session/list` (contains a project `cwd` in observed Air/OpenCode logs).
- Authentication calls (`authenticate` in v1, `auth/login` in v2), depending on the agent/client.

Do not assume every pre-project operation is stateless or safely repeatable. A request may cause authentication, create local state, emit unsolicited notifications, or depend on process-specific state.

**Proposed conservative MVP policy:**

- Forward and log requests while Agent A is active.
- Track all outstanding requests in **both directions**, including agent-initiated callbacks to the client.
- Treat the first project-bearing request as a handoff boundary. Do not forward it to Agent A until the decision is made.
- Do not switch while earlier requests are unresolved or while connection-scoped state is known to be nontransferable.
- Make the handoff policy explicit: replay only known-safe operations where needed, or fail with an actionable diagnostic. Never assume replaying authentication or arbitrary extension requests is safe.
- Capture and appropriately route or suppress notifications from the old agent after handoff. Never leak unexpected stale notifications or responses.

Authentication normally follows initialization, but initialized agents may already have credentials. Probe and project agents need access to the same necessary environment/configuration. Authentication or authorization performed only in Agent A is **not guaranteed** to carry over to Agent B.

## 7. Post-handoff proxying

After Agent B is ready, proxy ACP/JSON-RPC messages bidirectionally rather than implementing full agent behavior:

- Keep client-facing request IDs and internally routed request IDs correlated. Support overlapping/out-of-order responses.
- Support messages initiated by **either side** (filesystem calls, permissions, elicitation, etc.), not just client requests.
- Preserve JSON-RPC errors, notifications, cancellation, and extension fields.
- Route `session/*` operations to the correct backing process/session.
- Where multiple internal agents are supported, map session IDs and route requests/notifications accordingly, preventing collisions. For a single-process/single-project MVP this may not require changing session IDs, but routing still must be correct.
- Use correct stdio framing (newline-delimited JSON-RPC messages); never write diagnostic text to ACP stdout.
- Handle EOF, child exits, signals, backpressure, and clean termination of spawned processes.

Do not casually interpret or modify arbitrary JSON body fields. Only alter fields when needed by an explicitly supported feature.

## 8. Multiple projects on one client connection

ACP may allow multiple sessions associated with different project directories over one connection. A process has one launch CWD; therefore a naive switch-once proxy cannot transparently support unlimited project CWDs.

**MVP:** Support one project directory per client connection. If a later request identifies another project directory, reject clearly or have an explicit documented policy; do not silently send it to the wrong agent.

**Possible future extension:** Agent instance per project CWD, with per-agent initialization, session-ID routing, and multiplexed client/agent requests. This is a significant separate feature, not required for the first usable release.

## 9. Security and path handling

- Never accept an untrusted arbitrary path as a command. The executable and argument vector come from the user's explicit `-- COMMAND [ARG...]` configuration.
- Validate the selected CWD is an absolute existing accessible directory in the relevant host environment before spawning a project agent. Avoid silently falling back to HOME.
- Do not alter or broaden Docker mounts or project security policy. The underlying launcher remains responsible for mounts.
- Spawn the agent **without a shell** if possible, preserving executable/argument boundaries. Avoid `SYSTEM:`/shell quoting pitfalls from the diagnostic `socat` prototype.
- Preserve inherited environment by default, but avoid logging credentials, proxy headers, authentication tokens, or full sensitive ACP payloads unless explicitly enabled.
- Secure diagnostic file permissions and keep all diagnostics off stdout.

## 10. Suggested implementation

- **Language:** Go, consistent with `acp-wsl`; small cross-platform executable.
- **Transport:** stdin/stdout ACP framing using buffered readers/writers and safe handling of large messages.
- **Process API:** `exec.Command` with `Cmd.Dir=projectCwd`; inherit environment unless explicitly configured.
- **Core components:** client transport, child-agent transport, JSON-RPC request tracker, initialization recorder, CWD detector, agent lifecycle/handoff manager, diagnostics.
- **Logging:** optional structured diagnostic logs with invocation ID, event time, child PID, method, direction, response/error, selected CWD. Raw traffic logging must be opt-in and redactable.
- **CLI idea:** `acp-cwd [proxy options] -- COMMAND [ARG ...]`.
- **Exit behavior:** Report failures through JSON-RPC when there is a matching request; use stderr and exit status for transport/startup failures.

Do not use the previous shell-based stdio logger as production proxy logic; it was only a diagnostic aid.

## 11. Test matrix / acceptance criteria

### Minimum end-to-end tests

1. Start wrapper in an unrelated directory; client sends `initialize`, then `session/new` with project CWD. Assert project agent launches with **that** CWD and session works.
2. Repeat using `session/list` as the **first** project-bearing message.
3. Start wrapper directly in the project CWD; verify no unnecessary switch if optimization is implemented.
4. Verify full initialization request (including unknown `_meta`) is replayed faithfully.
5. Verify the client receives **one** initialization response even if two child processes are initialized.
6. Verify differing capabilities or incompatible protocol versions on Agent B result in a clear error, not silent protocol corruption.
7. Test request-ID reuse, simultaneous requests, agent-to-client callbacks and notifications around handoff.
8. Test authentication required before project discovery, as well as already-authenticated agents. Document unsupported transitions.
9. Test a second distinct project CWD on the same connection; enforce the chosen MVP policy.
10. Test launch failure, missing directory, early EOF, child crashes, client disconnect, SIGTERM, and process cleanup.
11. Confirm environment variables and argument arrays survive process launch; ensure no shell evaluation.
12. Confirm logging never contaminates protocol stdout and does not expose credentials by default.

### Real-world regression cases

- **JetBrains Air + aicage + OpenCode:** no fake config under Air's ACP directory; `session/new` succeeds using actual project config/mount.
- **JetBrains AI Assistant + aicage + OpenCode:** continue to work as before.
- **Air + `codex-acp` + `CODEX_PATH=aicage-codex`:** remain unaffected; optional comparison case, not necessary to proxy through `acp-cwd`.
- At least one non-aicage native ACP agent, to establish generic usefulness.

## 12. Open engineering questions (not yet resolved)

1. Which exact session/project-scoped ACP messages (v1 and v2) carry an authoritative `cwd`? Define a versioned method/path table rather than guessing by recursive JSON traversal.
2. Can the probe's advertised authentication, capabilities, and extensions reliably be reproduced in the project process? Define allowable differences and failure behavior.
3. What is the safe policy when authentication or another stateful pre-project operation has already occurred? Can the project agent reuse shared credential stores, or must the proxy block/re-authenticate?
4. Can a client issue agent-directed requests or receive agent notifications before a project is known that cannot safely be transferred?
5. What is the cleanest implementation of request/notification draining at the handoff boundary?
6. Should first release support only ACP v1, or also v2? The v2 schema and authentication flow differ; don't assume v1 JSON fields exist in v2.
7. Is a project-per-agent multiplexing mode worthwhile later, or is one project per proxy process sufficient for typical IDEs?
8. Should the proxy use the first agent as the actual project agent whenever possible, rather than always launching a second one?

## 13. Implementation order

1. Build tiny mock ACP clients/agents to reproduce Air-style launch behavior deterministically.
2. Implement lossless initialization forwarding and recording on Agent A.
3. Implement version-aware project CWD detection and Agent B spawn with `Cmd.Dir`.
4. Implement private initialization and validate Agent B's advertised contract.
5. Implement handoff and transparent bidirectional JSON-RPC routing with request correlation.
6. Add conservative handling of outstanding/stateful pre-project operations.
7. Add cleanup, security controls, diagnostics, and failure tests.
8. Run real Air/OpenCode/aicage regression test; document limitations clearly.

## 14. References

These are publicly accessible browser URLs to re-check when implementation starts; ACP specifications evolve.

- ACP v1 Initialization: https://agentclientprotocol.com/protocol/v1/initialization
- ACP v1 Authentication: https://agentclientprotocol.com/protocol/v1/authentication
- ACP v1 Session Setup: https://github.com/agentclientprotocol/agent-client-protocol/blob/main/docs/protocol/v1/session-setup.mdx
- ACP v1 Session List: https://agentclientprotocol.com/protocol/v1/session-list
- ACP v2 Initialization: https://github.com/agentclientprotocol/agent-client-protocol/blob/main/docs/protocol/v2/initialization.mdx
- ACP v2 Session Setup: https://github.com/agentclientprotocol/agent-client-protocol/blob/main/docs/protocol/v2/session-setup.mdx
- ACP v2 Migration: https://github.com/agentclientprotocol/agent-client-protocol/blob/main/docs/protocol/v2/migration.mdx
- codex-acp bridge: https://github.com/agentclientprotocol/codex-acp
- Related aicage tooling: https://github.com/aicage/acp-wsl

## 15. Bottom line

The idea is **not** to infer the project from IDE launch behavior or require user configuration per project. It is to let ACP reveal the project directory, then run the real agent **from that directory**, while preserving a coherent ACP connection for the IDE.

The unresolved technical challenge is **a safe, agent-independent handoff after initialization**. Implement and test that explicitly; do not assume that simply restarting a child process is transparent.
