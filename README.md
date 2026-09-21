# sanctum

An iPhone app meant to be the *only* app on the phone.

Sanctum is designed to run as the sole application in **Guided Access** (or MDM
Single App Mode). The phone boots, locks into Sanctum, and never leaves. There is
no home screen, no App Store, no browser, no feed. The device becomes a single
purpose-built tool instead of a general-purpose attention machine.

Because you cannot leave the app, Sanctum has to *contain* everything you
legitimately need from a phone. The base set:

| Surface | What it is |
| --- | --- |
| **Calendar** | Read/write the device calendars via EventKit. Day, week, agenda. |
| **Email** | A real mail client inside Sanctum — triage, read, reply. |
| **Claude** | Chat with Claude Code running on a sandbox this phone connects to. |
| **Messages** | WhatsApp and Signal, scoped to an explicit roster of people. |
| **Alarm** | Alarms and timers that actually fire, including from a locked-down phone. |

Claude and Messages share one **Talk** tab — both are conversations, and the
agent on your sandbox is one of the people you talk to.

Everything else is a deliberate addition, not a default.

## Status

Pre-code. Docs first — see [`docs/`](docs/).

- [Vision & principles](docs/vision.md) — why this exists and what it refuses to do
- [Architecture](docs/architecture.md) — app shape, modules, the sandbox backend
- [Notifications](docs/notifications.md) — what can interrupt you, and what should
- [Power](docs/power.md) — what burns it, and what plugging in all night costs
- [Guided Access](docs/guided-access.md) — how the phone gets locked down, and what that costs
- [Roadmap](docs/roadmap.md) — milestones and open questions
- Features: [calendar](docs/features/calendar.md) · [email](docs/features/email.md) · [claude](docs/features/claude.md) · [messaging](docs/features/messaging.md) · [alarm](docs/features/alarm.md)

## Repo layout (planned)

```
sanctum/
  docs/              # you are here
  Sanctum/           # iOS app target (SwiftUI)
  Packages/          # local SPM packages, one per feature domain
  Sandbox/           # relay/agent service that runs next to Claude Code
```
