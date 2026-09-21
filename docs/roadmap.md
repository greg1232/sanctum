# Roadmap

## M0 — Skeleton

Xcode project, `SanctumKit` with design system + `Clock` + Keychain, four-tab
shell with honest empty states, CI running tests and snapshot tests. No feature
logic. Runs on a real phone under Guided Access on day one so the constraints are
felt early.

Buy the Apple Developer Program on day one — push, AlarmKit and Time Sensitive
notifications all need it, and seven-day provisioning makes anything else a
waste of time. Build the provisioning-expiry self-check in the same pass; see
[building](building.md#the-expiry-landmine).

Wire up MetricKit from the first build — a long-running app should report its
own power behavior rather than be profiled after the fact. See [power](power.md).

Also blocking, and cheap: the 12-row
[notification verification matrix](notifications.md#verification-matrix), and
answering whether an individual can actually obtain an MDM push certificate —
that determines whether [Autonomous Single App Mode](guided-access.md#the-supervision-tax)
is available to us at all.

## M1 — Clock first

Alarm and timers on AlarmKit, plus the clock face. Chosen first because it's the
highest-reliability bar and the fastest path to "this phone is already useful on
a nightstand." AlarmKit is now the only alarm mechanism, so its verification is
part of the milestone rather than a follow-up.

## M2 — Calendar

EventKit, Today + 7-day agenda, event detail, quick add. Launch routing.

## M3 — Claude

`sanctumd` relay, Secure Enclave enrollment, streaming chat, tool activity,
approvals. Ship the sandbox service and the app side together.

## M4 — Email

IMAP/SMTP with OAuth, recent-window sync, threaded inbox, sanitized HTML
rendering, queued send. Longest milestone by a distance.

## M5 — Messages

`whatsmeow` + `signal-cli` inside `sanctumd`, the person-based roster, the Talk
tab merging Claude and messaging. Signal first — it's the supported linking flow
and the lower-risk account. WhatsApp second, behind a setup flow that states the
ban risk in words before you scan anything.

## M6 — Live-in

Two weeks of using it as the only app. Fix what actually breaks. Soak testing for
multi-day uptime. Then decide what's next from the gap list below, not from this
document.

---

## Open questions

Things we know we don't know. Each needs a decision before it needs code.

**Telephony.** No third-party app can place or receive normal calls, and SMS is
unreadable programmatically. Options: (a) accept the gap, (b) use CallKit +
a VoIP provider so Sanctum is a real phone over its own number, (c) put the SIM
in a different device. (b) is the honest answer and a large amount of work. This
is the biggest hole in "lose nothing you actually needed."

**iMessage.** Closed to us in every direction, with no companion-device seam to
exploit. Unlike WhatsApp and Signal (now planned for M5 — see
[messaging](features/messaging.md)), there is no route here at all. Accept it.

**Maps and transit.** MapKit works fine in-app. Turn-by-turn navigation on a
locked phone is a different question. Probably out of scope; probably the second
thing people miss.

**Photos and camera.** `PHPicker` gives access to the library; capture needs our
own camera UI. Worth it only if mail attachments demand it.

**Music / podcasts / audiobooks.** Nothing plays audio on this phone unless
Sanctum plays it. A local-files player is a weekend; streaming services are not
accessible. Bluetooth from another device sidesteps it entirely.

**SMS for 2FA.** The practical sting of the telephony gap: no SMS means no
login codes. TOTP where offered, an Android SMS-forwarder or a VoIP number
otherwise. Probably decided together with telephony.

**Notes / capture.** Arguably already covered by Claude on the sandbox, which is
a better scratchpad than a notes app. Worth testing before building anything.

**Pocket money.** Wallet still works under Guided Access, so payments are handled
by the system. No work needed — confirm this on a real device.

---

## Candidate features (post-v1)

Not commitments. Ordered by how much they exploit the fact that Sanctum owns the
whole device — the ones near the top are things a normal app simply cannot do.

1. **Bedtime mode.** At a set hour the app collapses to the clock face; mail and
   chat are unavailable until the morning alarm. Containment as a feature.
2. **Daily brief.** One screen on first unlock: today's events, what needs a
   reply, what the agent finished overnight. Finite by construction.
3. **Modes.** Work / evening / weekend change which tabs exist at all. The phone
   is a different object at 9am and 9pm.
4. **Agent-scheduled work.** Ask Claude to do something overnight on the sandbox;
   see the result in the morning brief.
5. **Focus sessions.** A timer that hides every tab but one until it expires.
6. **Local-files audio player.** Nightstand use: white noise, audiobooks.
7. **Offline reading queue.** Links saved from mail, fetched and stripped by the
   sandbox, read in-app — a narrow, finite replacement for the browser seam.
8. **Weather.** Small, expected, cheap. Deliberately last: it's the kind of
   feature that's easy to build and easy to let sprawl into a feed.
