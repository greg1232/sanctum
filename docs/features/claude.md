# Claude

A chat surface that talks to **Claude Code running on a sandbox you control**.
Not a wrapper around a chat API — the thing on the other end is an agent with a
filesystem, a shell, and your repos.

This is what makes Sanctum more than a nice minimal phone: the locked-down device
is a terminal onto a machine that can actually do work.

## Topology

```
 iPhone (Sanctum)            your sandbox (VM / home server / cloud box)
 ┌───────────────┐   TLS     ┌──────────────────────────────────────┐
 │ Chat UI       │◄────────► │  sanctumd (relay)                    │
 │ device key    │  WebSocket│    ├── session manager               │
 │ local cache   │           │    ├── Claude Code / Agent SDK       │
 └───────────────┘           │    └── transcript store              │
                             └──────────────────────────────────────┘
```

`sanctumd` lives in `Sandbox/` in this repo. Responsibilities:

- Terminate TLS, authenticate the device, and nothing else on the public edge.
- Own Claude Code sessions: start, resume, cancel, and keep them alive across
  phone reconnects.
- Persist transcripts server-side so a lost or reset phone loses no history.
- Stream output incrementally (tokens, tool-use events, results).

Explicitly **not** doing: multi-user support, a web UI, any hosted component we
operate. One person, one sandbox.

## Transport

WebSocket over TLS, newline-delimited JSON frames. Chosen over plain SSH because
we want structured tool-use events and resumable sessions, not a terminal
emulator on a 6-inch screen.

Frame kinds (sketch, not final):

```
→ {"t":"hello","device":"<pubkey-id>","proto":1}
→ {"t":"prompt","session":"<id>","text":"..."}
→ {"t":"cancel","session":"<id>"}
← {"t":"delta","session":"<id>","text":"..."}
← {"t":"tool","session":"<id>","name":"Bash","status":"running","summary":"..."}
← {"t":"done","session":"<id>","stop":"end_turn"}
← {"t":"error","code":"...","message":"..."}
```

Reconnect resumes by session id and replays anything missed since the last
acknowledged frame.

## Auth

- Device generates a keypair in the **Secure Enclave** on first run.
- Enrollment is a one-time out-of-band step: a short-lived code printed by
  `sanctumd` and typed into the phone, which binds the public key.
- Every connection is signed. No passwords, no bearer token in a config file.
- One device key may be revoked on the sandbox side without touching the others.
- `sanctumd` binds to loopback by default; exposure is the operator's explicit
  choice (Tailscale/WireGuard strongly preferred over a public port).

## Surfaces

1. **Conversation.** Streaming replies, markdown rendered, code blocks with
   horizontal scroll and copy. Monospace where it matters.
2. **Tool activity.** Collapsed one-liners ("Read src/main.swift", "Ran tests —
   passed") that expand on tap. You should be able to follow what the agent is
   doing from a phone without drowning in output.
3. **Approvals.** When Claude Code needs permission for a tool call, the phone
   shows what it wants to do and Allow / Deny. This is the load-bearing UI of the
   whole feature — an agent on a box you own, approved from your pocket.
4. **Sessions.** A short list of recent conversations, resumable. Not a
   searchable archive in v1.

## Honest failure

The sandbox will be unreachable sometimes. When it is: say so at the top of the
tab, show cached transcripts read-only, let the user queue nothing (a queued
prompt to an agent is a surprise, not a convenience), and retry with backoff.

Per [architecture](../architecture.md#offline--online-split), no other feature
may degrade when this one is down.

## Open questions

- Voice input. Dictation is native and probably enough; push-to-talk with local
  transcription is the ambitious version.
- Notifications when a long-running agent turn finishes. Resolved in
  [power](../power.md#the-correction-no-relay-is-needed): `sanctumd` is the APNs
  provider directly, so this costs a `.p8` key rather than new infrastructure.
  Open only in the sense of taste — an agent that pings on every tool call is a
  feed, so default to notifying on turn completion and approval requests only.
