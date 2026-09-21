# Notifications & interruption

Two separate questions that get conflated:

1. **What can technically reach you** on a phone pinned to one app.
2. **What should** — which is the design question, and the interesting one.

## The reframe: Sanctum is foreground, so it mostly doesn't need notifications

On a normal phone, a notification is how an app that isn't running gets your
attention. Sanctum *is* running. It is the foreground app, and on the
recommended nightstand configuration (auto-lock off) it is the foreground app
continuously, for days.

That collapses most of the problem:

- Sanctum doesn't need a banner to tell you a message arrived. It can render it.
- It doesn't need push to know something happened. Its WebSocket to
  [`sanctumd`](features/claude.md) is already open.
- What it needs isn't notification, it's **attention routing**: something
  happening in one tab pulling focus while you're in another. That's an in-app
  concern — a router and a presentation policy — not APNs.

**APNs is only needed for the configuration where the screen locks** — and we've
since decided the screen *should* lock. See
[power](power.md#sleep-and-wake--the-screen-sleeps-always): an always-on display
quietly turns Sanctum into a stationary appliance, so the screen sleeps like any
phone's and Sanctum draws no persistent clock at all.

So push exists. It is much cheaper than this document originally implied: APNs
token auth means **`sanctumd` is the provider directly** — a `.p8` key and an
HTTP/2 client on a box you already run, no relay and nothing hosted. The real
prerequisite is a paid Apple Developer account, which AlarmKit and Time Sensitive
notifications need anyway.

Push payloads pass through Apple, so they carry minimal text ("Sam sent a
message") and the app fetches real content over its own socket on wake.

## Layer 1 — Sanctum itself

Always works. No restrictions, because there's nothing to break into; it's the
app on screen.

- Rendering anything, anywhere in the UI.
- Sound via `AVAudioSession` — an active session plays regardless of banners.
- Haptics via Core Haptics.
- Full-screen takeover of its own UI (ringing alarm, imminent event).
- **AlarmKit** (iOS 26+) for real alarms: breaks through silent mode and Focus
  with a system-level full-screen alert. The sanctioned break-through mechanism,
  and why it's the primary alarm implementation — see [alarm](features/alarm.md).
- Local `UNNotification`s, though with the app foregrounded these mostly serve as
  a fallback for the auto-lock-on configuration.

## Layer 2 — System-level, cannot be suppressed

These reach you regardless of Guided Access, and regardless of anything Sanctum
does. Worth knowing precisely, because they are the *only* things that can
interrupt from outside.

- **Wireless Emergency Alerts.** National/Presidential alerts cannot be disabled
  by the user at all. AMBER and extreme-weather alerts can be toggled in
  Settings but not suppressed by an app. These will take over the screen.
- **Emergency SOS.** The side + volume hold gesture works during Guided Access
  and Sanctum cannot and must not interfere. See
  [guided-access.md](guided-access.md#emergency-access).
- **Exiting Guided Access itself.** Triple-click + passcode/Face ID. Always
  available.

## Layer 3 — Other apps — the part that needs verifying on a device

This is where I'd rather not assert from memory. The generally-understood
behavior of Guided Access is that it suppresses other apps' notification banners
and blocks Notification Center and Control Center entirely — that's precisely why
it's used for kiosks. But the specifics have varied across iOS versions, and
Guided Access and MDM Single App Mode do not behave identically.

Several of these cut both ways for us. An incoming call breaking through is a
*hole* in the lockdown and simultaneously a partial answer to the telephony gap
in [roadmap](roadmap.md#open-questions). We should know which world we're in
before designing around either.

### Verification matrix

Run on a real device, both Guided Access and Single App Mode, and record the
answer next to each row. Until then, treat every one as unknown.

| # | Stimulus | Question | Result |
| --- | --- | --- | --- |
| 1 | Notification from another installed app | Banner shown? Sound played? | ? |
| 2 | Incoming cellular call | Call UI takes over? Can it be answered? | ? |
| 3 | Incoming CallKit VoIP call | Same, from a third-party app | ? |
| 4 | Apple Clock app alarm | Fires? Full-screen? Dismissable? | ? |
| 5 | Sanctum AlarmKit alarm | Fires through silent + Focus? | ? |
| 6 | Sanctum local notification, app foregrounded | Presented? | ? |
| 7 | Low Battery warning | System alert appears? | ? |
| 8 | Charging connect/disconnect chime | Sound plays? | ? |
| 9 | AirPods / Bluetooth connection banner | Shown? | ? |
| 10 | Emergency/WEA test alert | Takes over? (assumed yes) | ? |
| 11 | Focus mode change | Does it affect Sanctum's own audio? | ? |
| 12 | App crash or jetsam kill | Relaunch, black screen, or home screen? How long? Researched: undocumented, reports say up to an hour. | ? |
| 12b | Mirror Display Auto-Lock on + short Auto-Lock | Does the screen sleep on the system schedule? | ? |
| 12c | Soak: session running 7+ days untouched | Do the two-day degradation reports reproduce? | ? |

### Background services matrix

Same exercise for the claim in
[guided-access](guided-access.md#what-the-lockdown-does-not-touch) that Guided
Access leaves other apps' background modes alone. It's load-bearing for whether
the phone can still open your car, so measure it rather than trust it.

| # | Stimulus | Question | Result |
| --- | --- | --- | --- |
| 13 | Walk up to a Tesla with phone key paired | Does the car unlock? | ? |
| 14 | Wallet car/home/transit key, Express Mode | Works held to a reader? | ? |
| 15 | Apple Pay, double-click, **Sleep/Wake enabled in Options** | Does Wallet appear over the pinned app? (Confirmed blocked with it off.) | ? |
| 15c | Apple Pay via AssistiveTouch | Apple's suggested workaround — does it complete? | ? |
| 15b | Wallet boarding pass or barcode | Reachable at all without exiting? | ? |
| 16 | Apple Watch | Stays connected, notifications relay to it? | ? |
| 17 | AirPods | Auto-connect on open? | ? |
| 18 | CarPlay, wired and wireless | Connects? What shows on the car screen? | ? |
| 19 | Siri, side-button hold | Invocable? Can it leave the app or act? | ? |
| 20 | Shortcuts personal automation (charger disconnect) | Does it fire during a session? | ? |

Rows 19 and 20 are power and capability questions as much as notification ones —
see [power](power.md#questions-to-settle).

Row 16 is quietly interesting: if the Watch keeps relaying, it becomes a second
notification channel that sidesteps most of this document.

Row 12 is the one people forget. If a crash drops the phone to the home screen
and the Guided Access session ends, the whole premise fails silently overnight.
Single App Mode relaunches; Guided Access is the one to check.

## What *should* break in

The design question. Because Sanctum is the only thing that can reach you, two
things are true at once, and they pull in opposite directions:

- **Every dropped notification is total.** On a normal phone, if one app fails
  you'd still catch it somewhere else. Here there is no second channel. Missing
  something is a real failure, not a minor bug.
- **Every notification is trusted.** There is no marketing, no engagement
  growth-hacking, no app trying to reacquire you. The roster model means a
  message is from someone you explicitly chose. So the interruption budget can
  be *spent*, rather than defended.

### Proposed policy

| Source | Interrupts | Rationale |
| --- | --- | --- |
| Alarm / timer firing | **Always.** Full-screen takeover from any tab. | The one thing that must never be missed. |
| Event starting ≤15 min | **Yes.** Pulls to Calendar on launch, banner in-app otherwise. | Time-critical and finite. |
| Roster message | **Yes.** In-app banner, sound, haptic. | Chosen people. That's what the roster is for. |
| Non-roster message | **Never.** Quarantined silently. | [messaging](features/messaging.md#inbound-from-people-not-on-the-roster) |
| Claude finished a turn | **Configurable, default yes.** | You asked it to do something. But an agent that pings on every tool call is a feed. |
| Claude needs approval | **Yes.** | It's blocked on you. This is the load-bearing case. |
| New email | **Never.** | Email is a pull surface. Nothing arriving by email is urgent enough to interrupt; if it is, that person is on your roster. |
| Sandbox disconnected | **No.** Persistent inline state, not an alert. | It'll happen. Alerting on it trains you to ignore alerts. |

The email row is the opinionated one and the most likely to be argued with. My
position: the moment email can interrupt, the phone is a notification device
again, and the roster stops meaning anything.

### Rules

- **No badges anywhere.** A number that only ever goes up is a feed with extra
  steps. Unread state is visible when you open a tab, not from the tab bar.
- **No notification is ever a lie.** If Sanctum shows it, it's real and current.
  No optimistic "sent" that hasn't sent, no stale count.
- **Night is quieter.** Between bedtime and the morning alarm, only alarms and
  approvals break through. The rest waits until you're awake — one of the things
  owning the whole device makes trivially possible.

## Open questions

- **Verification matrix, all 12 rows.** Blocking work for M0. Cheap to run,
  invalidates real design if it comes out the other way.
