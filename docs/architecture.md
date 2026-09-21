# Architecture

## Shape

One iOS app target, four feature domains, one optional backend.

```
┌──────────────────────── iPhone (Guided Access) ────────────────────────┐
│                                                                        │
│   Sanctum.app  (SwiftUI, iOS 26+)                                      │
│   ┌──────────┬──────────┬──────────┬──────────┐                        │
│   │ Calendar │  Email   │  Claude  │  Alarm   │   ← feature packages   │
│   └────┬─────┴────┬─────┴────┬─────┴────┬─────┘                        │
│        │          │          │          │                              │
│   ┌────┴──────────┴──────────┴──────────┴─────┐                        │
│   │ SanctumKit: design system, storage,       │                        │
│   │ keychain, networking, logging, clock      │                        │
│   └────┬──────────┬──────────┬──────────┬─────┘                        │
│        │          │          │          │                              │
│    EventKit    IMAP/JMAP   WebSocket  AlarmKit                         │
│        │          │          │          │                              │
└────────┼──────────┼──────────┼──────────┼──────────────────────────────┘
         │          │          │          │
   system cals   mail host   sandbox    (on-device)
                            (yours)
```

## Layering rules

- **Feature packages never import each other.** Cross-feature interaction goes
  through small protocols declared in `SanctumKit` (e.g. `CalendarReading`, so
  the Claude surface can answer "what's on today" without linking Calendar).
- **`SanctumKit` imports no feature.** It owns the design system, keychain,
  persistence, an injectable `Clock`, network primitives, and logging.
- **Everything network-facing is behind a protocol with a fake.** The whole app
  must run in previews and tests with no sandbox, no mail host, no calendar
  permission.
- **No feature may be required for another to launch.** The app opens with
  whatever is available and shows honest empty/error states for the rest.

## Offline / online split

| Domain | Works with no network | Needs network | Needs the sandbox |
| --- | --- | --- | --- |
| Calendar | ✅ full (EventKit local store) | sync from providers | no |
| Alarm | ✅ full | no | no |
| Email | ✅ read cached window, queue outbound | fetch/send | no |
| Claude | ❌ | ✅ | ✅ |

The sandbox is a single point of failure for exactly one surface, by design.
Losing it must never cost you an alarm or a meeting.

## Navigation model

A fixed four-item tab bar. No nested tabs, no hamburger, no hidden drawers. Depth
is capped at three pushes (list → item → compose/edit). Anything deeper is a
design failure.

Because there's no home screen, **launch state matters**: Sanctum opens on a
context-appropriate surface (a ringing alarm, an event starting within 15 min,
otherwise the last tab) rather than always resetting to the same place.

## Platform target

- iOS 26+, SwiftUI-first, Swift 6 strict concurrency.
- iPhone only, portrait only. No iPad or Mac Catalyst — the lockdown story is
  different on each and we don't want to owe three of them.
- No third-party dependencies in the app target unless a protocol implementation
  is genuinely unreasonable to hand-roll (candidate: IMAP). Every dependency gets
  a written justification in the PR.

## The sandbox backend

A small service you run on a machine you control (VM, home server, cloud box),
sitting next to a Claude Code installation. See
[docs/features/claude.md](features/claude.md) for the protocol. It is:

- **Yours.** No Sanctum-operated infrastructure exists. There is no fleet.
- **One user.** Auth is a device keypair enrolled once over a trusted channel.
- **Stateless-ish.** Conversation history lives on the sandbox (so a lost phone
  loses nothing but its session), with a local cache on device for offline read.

## Persistence

- Mail cache + Claude transcript cache: SQLite via GRDB or SwiftData — decision
  deferred to the first real schema, not before.
- Credentials (mail OAuth/app passwords, sandbox device key): Keychain,
  `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`, no iCloud sync.
- Preferences: `UserDefaults`, but read through a typed wrapper in `SanctumKit`.

## Testing

- Unit tests per package, no network, deterministic `Clock`.
- Snapshot tests on the design system in light/dark and at the largest Dynamic
  Type size — a locked-down phone is often used at arm's length on a nightstand.
- One end-to-end harness that boots the app against fakes for all four domains
  and walks the core flows.
