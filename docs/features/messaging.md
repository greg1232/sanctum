# Messaging

A client for talking to **specific people**. Not an inbox.

This is the design decision everything else follows from. Sanctum does not ship a
general-purpose WhatsApp or Signal client with your full contact list, chat
archive, groups, and a search-for-anyone field. It ships a short, explicit roster
of people you actually talk to, and it can reach each of them over whatever
network they're on.

It's the [vision](../vision.md#principles) applied to messaging: a finite list,
bounded by a real-world quantity — the number of people in your life.

## People, not accounts

The core model is a **person**, defined in configuration, with one or more
addresses:

```yaml
people:
  - name: Sam
    signal: "+1555..."
    whatsapp: "+1555..."
    email: sam@example.com
    prefer: signal
  - name: Mom
    whatsapp: "+1555..."
    prefer: whatsapp
```

Consequences that fall out of this, all good:

- **One thread per person**, not per network. Sam's Signal and WhatsApp messages
  are one conversation in one place. Sending picks `prefer`, or whatever they
  last used; the network is a small label, not a mode you're in.
- **No contact sync, no discovery, no search.** There is no UI for "message
  someone new." Adding a person is an act of configuration you perform
  deliberately, not a thing you do at 11pm.
- **Groups are opt-in and named.** A group is just another roster entry pointing
  at a group id. Groups you haven't listed do not appear. This alone removes most
  of the volume and most of the complexity.
- **The roster is the notification policy.** Everyone on it can reach you;
  nobody else can. No per-thread mute settings, because there's nothing to mute.

### Inbound from people not on the roster

Real messages from unlisted senders will arrive, and dropping them silently is
how you miss something that mattered.

Default: **quarantine, don't notify.** Non-roster messages land in a single
collapsed row at the bottom of the list — "4 messages from people not on your
list" — readable on purpose, never pushed, never badged. Promoting a sender to
the roster is one tap and takes effect immediately; replying without promoting
isn't possible.

This is the one policy I'd most expect you to want to change. The alternatives
are notify-but-quarantine (safer, leakier) and hard-drop (cleaner, riskier).

## How it connects

Neither network has a client API for personal accounts, and no iOS app can read
another app's messages. But both support **linked / companion devices** — that's
how WhatsApp Web and Signal Desktop work.

So the phone is not a WhatsApp client. The sandbox we already run for
[Claude](claude.md) is. The phone is a screen for it.

```
 iPhone (Sanctum)                  your sandbox
 ┌──────────────┐             ┌──────────────────────────────────┐
 │  Talk tab    │◄─── TLS ───►│  sanctumd                        │
 │              │  one ws,    │   ├── Claude Code sessions       │
 │  roster      │  one proto  │   ├── roster + normalized store  │
 │  local cache │             │   ├── whatsmeow  (companion)     │
 └──────────────┘             │   └── signal-cli (linked device) │
                              └──────────────────────────────────┘
```

**Decision: a thin bridge inside `sanctumd`, not Matrix.**

Standing up a Matrix homeserver with `mautrix-whatsapp` and `mautrix-signal` is
the standard answer here, and it's the right one if you want full fidelity across
arbitrary chats — groups, history, reactions, every network under the sun. A
roster-scoped client wants none of that. We'd be running four moving parts
(homeserver, two bridges, a client SDK) and inheriting Matrix's cross-signing and
key-backup UX — which is genuinely painful to debug on a phone locked into one
app — to get normalization we mostly throw away.

Instead: `whatsmeow` (Go, the same library the WhatsApp bridge is built on) and
`signal-cli` in JSON-RPC daemon mode, both feeding one normalized store in
`sanctumd`, exposed over the **WebSocket frame protocol we already designed for
Claude**. One connection from the phone, one auth story, one cache.

Scope that makes this small: 1:1 and listed groups, text + images + voice notes,
replies and reactions rendered. No arbitrary chat sync, no full history import,
no presence.

**Reconsider if** the roster grows past ~30 people, groups become central, or we
want a third and fourth network. At that point the bridges earn their keep and
`matrix-rust-sdk`'s Swift bindings become the cheaper client path.

## Per-network specifics

### Signal

Good posture. Linking a secondary device is a **first-class, supported feature**,
and `signal-cli` rides that official flow. Third-party clients aren't blessed, but
signal-cli has been in wide use for years with no enforcement story. Setup is a QR
scan from your phone, once.

Newly linked devices receive limited prior history — expect the archive to start
the day you link. `signal-cli` can alternatively register the number as a
*primary* on the sandbox, which is architecturally cleaner but costs you Signal on
every other device; only sensible with a dedicated number.

### WhatsApp

Worse posture, stated plainly.

- Links as a **companion device** via QR or pairing code, like WhatsApp Web.
- **You still need a primary.** The account stays registered on a real phone
  running the official app, and that phone must check in periodically (roughly
  every two weeks — verify against current behavior) or companions get logged
  out. In practice: an old handset on wifi in a drawer.
- **This violates WhatsApp's terms.** Meta does ban accounts for unofficial
  clients. Companion linking is far less conspicuous than faked primary
  registration, and a roster of ten people is not the volume their abuse
  detection hunts — but the honest version is: **you may lose your number.**
  Don't do this with an account you can't afford to lose.

The roster model hedges this nicely. If WhatsApp goes away, people with a
`signal:` address keep working and the thread survives.

### SMS — the one that actually bites

Not WhatsApp or Signal, same tab, sharper problem: **two-factor codes**. A phone
that can't receive SMS can't log into things. Routes: an old Android handset
running an SMS-forwarding bridge, a VoIP number with an API, or moving 2FA to
TOTP wherever it's offered. Ties into the telephony question in
[roadmap](../roadmap.md#open-questions) and is probably answered with it.

## What this costs you

**End-to-end encryption now ends at the sandbox, not at your phone.** A linked
device decrypts your messages — that's what it's for — which means plaintext
passes through, and history sits at rest on, a server. If that box is
compromised, so is everything you've said on both networks.

Mitigations, none of which fully fix it: full-disk encryption on the sandbox;
never exposing it publicly (Tailscale/WireGuard only, per
[claude.md](claude.md#auth)); a short retention window on the normalized store,
since the roster model means you don't need an archive anyway.

And the part that isn't about you: everyone on your roster chose an E2EE app.
This weakens that for them, without their knowledge. Worth sitting with before
building it.

## Where it lives in the app

Adding a Messages tab makes five, and [architecture](../architecture.md#navigation-model)
says four. Rather than growing the tab bar, **merge Claude and Messages into one
"Talk" tab**: a single roster where Claude is one entry, pinned at the top.

This is better than a compromise. Both are streaming conversations with the same
shape, and the agent running on your sandbox genuinely is one of the people you
talk to. The list stays finite and the principle survives intact.

Per-thread: standard bubbles, media inline, reactions and replies rendered.
No typing indicators, and **no read receipts sent** — a locked-down phone is a
good excuse to stop broadcasting presence.

## Open questions

- **Notifications.** Settled: `sanctumd` pushes via APNs directly (see
  [power](../power.md#the-correction-no-relay-is-needed)), with minimal payloads
  — "Sam sent a message", never the body, since push passes through Apple. Still
  open is whether per-sender rules are worth it; a ten-person roster is a volume
  where they'd actually be tractable.
- **Non-roster policy.** Quarantine vs. notify vs. drop (above).
- **Media at rest.** Photos and voice notes proxied and cached where, for how
  long, under what key.
- **Voice notes.** Very common on WhatsApp. Playback is easy; recording and
  sending is a real piece of UI.
- **Calls.** Both networks carry voice and video. Linked devices don't bridge
  them. Out of scope — and worth *saying* so in the UI rather than silently
  dropping an incoming call.
