# Architecture

## Shape

One iOS app target, four feature domains, one optional backend.

```
┌──────────────────────── iPhone (Guided Access) ────────────────────────┐
│                                                                        │
│   Sanctum.app  (SwiftUI, iOS 26+)                                      │
│   ┌──────────┬──────────┬─────────────────────┬──────────┐            │
│   │ Calendar │  Email   │        Talk         │  Alarm   │  ← packages │
│   │          │          │  Claude · Messages  │          │             │
│   └────┬─────┴────┬─────┴──────────┬──────────┴────┬─────┘            │
│        │          │                │               │                   │
│   ┌────┴──────────┴────────────────┴───────────────┴─────┐            │
│   │ SanctumKit: design system, storage,                  │            │
│   │ keychain, networking, logging, clock                 │            │
│   └────┬──────────┬────────────────┬───────────────┬─────┘            │
│        │          │                │               │                   │
│    EventKit    IMAP/SMTP      WebSocket        AlarmKit                │
│        │          │                │               │                   │
└────────┼──────────┼────────────────┼───────────────┼───────────────────┘
         │          │                │               │
   system cals   mail host        sanctumd         (on-device)
                              (yours: Claude Code,
                               whatsmeow, signal-cli)
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

## Put changeable logic on the sandbox

The two halves of this system deploy on wildly different terms, and it should
shape where code lives.

- **`sanctumd` deploys in seconds.** SSH, replace a binary, restart. As often as
  you like.
- **The app deploys with physical access.** Exit Guided Access, plug into a Mac,
  build, install, re-lock. And it is signed with a certificate that
  [expires](building.md#the-expiry-landmine).

So: **when a piece of behavior could plausibly live on either side, put it on the
sandbox.** Rules, rosters, schedules, credentials, integrations and policy are
all things you will want to change at 11pm without a cable. The app should be a
good client — rendering, input, and the things that must work with no network at
all — and as little policy as possible.

This retroactively explains several decisions already made: the messaging roster
is sandbox configuration rather than app state, integrations are
[agent tools rather than tabs](integrations.md#the-pattern-integrations-are-agent-tools-not-tabs),
and Claude transcripts live server-side so a reinstalled phone loses nothing.

The limit is the [offline split](#offline--online-split) below: anything that
must work without the sandbox cannot live on it. Alarms and calendar are
on-device precisely because you cannot ship a fix to a phone that failed to wake
you.

## Offline / online split

| Domain | Works with no network | Needs network | Needs the sandbox |
| --- | --- | --- | --- |
| Calendar | ✅ full (EventKit local store) | sync from providers | no |
| Alarm | ✅ full | no | no |
| Email | ✅ read cached window, queue outbound | fetch/send | no |
| Claude | ❌ | ✅ | ✅ |
| Messages | ✅ read cached threads | ✅ | ✅ |

The sandbox is a single point of failure for exactly one surface, by design.
Losing it must never cost you an alarm or a meeting.

## Navigation model

A fixed four-item tab bar — Calendar, Email, Talk, Alarm. Claude and Messages
share the Talk tab rather than growing the bar to five; see
[messaging](features/messaging.md#where-it-lives-in-the-app). No nested tabs, no
hamburger, no hidden drawers. Depth
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
- **More than Claude.** It also hosts the WhatsApp and Signal companion clients
  and the normalized message store — one service, one socket, one auth story.
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
