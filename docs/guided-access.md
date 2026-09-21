# Guided Access

Sanctum's premise is that the phone cannot leave the app. iOS offers three
mechanisms for this. They are not equivalent, and the differences decide what
kind of object the phone becomes.

## A — Guided Access

Accessibility → Guided Access. Triple-click the side button inside Sanctum to
start a session; triple-click plus passcode or Face ID to end it.

- **Pros:** zero infrastructure, works on any iPhone in about 90 seconds, trivial
  to exit in an emergency, nothing is erased.
- **Cons:** it's a *session*, not a device state. **It does not survive a
  reboot** — the phone comes up on the home screen and a human has to re-enter
  Guided Access by hand. Also changes the Home indicator and some system gestures
  in ways that affect our UI (see below).

Default for development and for "my own phone, evenings." What the rest of these
docs assume unless stated.

## B — Autonomous Single App Mode (ASAM)

The app locks *itself*. Sanctum calls `UIAccessibility.requestGuidedAccessSession(enabled:)`
and iOS pins the device to it — no triple-click, no Accessibility menu. It works
only if the device is **supervised** and an MDM payload lists Sanctum's bundle ID
as permitted for autonomous mode. This is the mechanism behind school exam apps.

- **Pros:** programmatic. Sanctum decides when the lock is on and when it lifts.
- **Cons:** still needs supervision and MDM to authorize. Still doesn't survive
  reboot, because after a restart nothing has launched Sanctum to make the call —
  though recovery is one tap on an icon rather than a trip through Settings.

**This is the most interesting option for Sanctum**, and the docs undersold it.
A lock the app controls turns containment into something dynamic rather than a
switch someone flips: bedtime mode can tighten the lock and the morning alarm can
release it; a genuine emergency path can lift it without a passcode dance;
[modes](roadmap.md#candidate-features-post-v1) become real rather than
cosmetic. The risk is the obvious one — a bug in our code is now a bug in the
lock — so the release path needs to be simple enough to audit in one sitting.

## C — Single App Mode (device-level app lock)

An MDM payload (`com.apple.app_lock`) pinning a supervised device to one bundle
ID as **device state**, not a session. The phone boots directly into Sanctum.
There is no home screen to return to.

- **Pros:** survives reboot, survives update. This is the only option that makes
  the phone genuinely an appliance.
- **Cons:** the heaviest setup. Supervision requires erasing the device, and
  lifting the lock means changing the payload or un-enrolling.

Right answer for a dedicated second phone. Overkill for a primary.

## The supervision tax

B and C both require **supervision**, and that's the real barrier, so be clear
about what it costs:

1. **Erase the device.** Supervision is applied at setup, via Apple Configurator
   on a Mac. There's no supervising a phone in place.
2. **Get an MDM.** Something has to push the payload. Options run from hosted
   (Mosyle, Jamf) to self-hosted open source (NanoMDM, MicroMDM) — the latter
   being a natural fit given we already run a sandbox.
3. **Get an APNs certificate for MDM.** ⚠️ **Verify this before planning around
   B or C.** An MDM push certificate is obtained by submitting a CSR signed by an
   MDM *vendor* certificate, which historically meant the Apple Developer
   Enterprise Program. Community services exist that sign CSRs for self-hosted
   MDM users, and hosted MDMs handle it for you. I'm not certain of the current
   requirements for an individual, and it's the step most likely to block this
   route entirely.

Whether Apple Configurator alone can apply the app lock without a full MDM server
is also worth checking — it would remove step 3 outright and make C dramatically
more accessible.

## Recommendation

Build against **A**, design for **B**. Guided Access costs nothing and is enough
to develop and live with. Autonomous Single App Mode is what Sanctum should
ultimately want, because a lock the app itself controls is a feature rather than
a constraint — but it should be an upgrade path, never a requirement to run.

Nothing in the app may assume it is locked down at all. Sanctum has to be a
decent app on an ordinary unsupervised phone, or it can't be developed.

## Session options you must change

⚠️ **Guided Access's defaults are wrong for Sanctum in two ways**, and both were
found by research rather than assumption. Neither is the default, and missing
either one silently breaks a decision made elsewhere in these docs.

### 1. Auto-lock is disabled by default — this is the big one

**A Guided Access session keeps the screen on indefinitely.** The system
Auto-Lock setting stops applying the moment the session starts. This has been
the behavior since iOS 9 and is well documented by kiosk operators, who hit it
as battery drain on unattended devices.

That directly contradicts [the decision to let the screen sleep](power.md#sleep-and-wake--the-screen-sleeps-always)
and would silently reinstate the always-on profile we removed — roughly **24% of
the battery per night**, with none of the benefit, since Sanctum wouldn't even be
drawing a clock.

The fix is a setting called **Mirror Display Auto-Lock**, in
Settings → Accessibility → Guided Access. Turning it on makes the session respect
the system Auto-Lock. Newer iOS also exposes a **Display Auto-Lock** timeout
directly in the session's Options sheet.

**Turn this on. Everything in [power](power.md) assumes it.** Confirm the exact
name and location on iOS 26 — this has moved between versions.

### 2. The side button is disabled by default

In the Options sheet shown before starting a session, hardware buttons can be
individually enabled, and **Sleep/Wake defaults to off** — the side button is
simply ignored for the duration.

Turn it on. Two reasons: you get manual lock (press to sleep, rather than waiting
out the timeout), and it's a precondition for Apple Pay working at all — see
below.

Note this is configured **per session, per app**, so it has to be re-set every
time a session is started rather than once globally. On a phone that reboots,
that's a step you will forget. It's a genuine argument for
[Autonomous Single App Mode](#b--autonomous-single-app-mode-asam), where the app
requests its own session and the configuration doesn't depend on someone
remembering an Options sheet.

## What the lockdown does *not* touch

The mental model that matters:

> **Guided Access constrains the screen, not the radios.**

It restricts which app *you* can interact with. It does not change the process
lifecycle of other apps, and it does not disable their background modes. Anything
on the phone that already works with the device in your pocket keeps working,
because it never needed the foreground in the first place.

### Worked example: a Tesla phone key

Tesla's phone key is BLE. You walk up, the car unlocks, and you never open the
app — that's the entire design. It works from a backgrounded, suspended app via
iOS's Bluetooth background modes, with the key material in the Secure Enclave.

So under Guided Access it should simply keep working. **The key works; the app
doesn't.** What you lose is everything behind Tesla's UI — climate, charge
limits, summon, checking state of charge — because that genuinely needs the
foreground, and Guided Access denies it.

What this class of thing needs from us is nothing, except discipline about setup:
pairing a phone key requires the Tesla app in the foreground and the key card on
the console. That has to happen **before** the phone is locked down.

### The general category

Things that should survive the lockdown untouched, all for the same reason:

| Still works | Why |
| --- | --- |
| Tesla / BLE phone keys | Bluetooth background modes, no UI needed |
| **Express Mode** Wallet items — transit, car, home and hotel keys | The secure element answers the reader with no UI at all. See below. |
| Apple Watch pairing and connection | Background BLE |
| AirPods auto-connect | Background BLE |
| Find My / AirTag network participation | System-level |
| Location-based system services | Background modes are unaffected |

And the flip side — things that need another app's UI, and are therefore gone:
the Tesla app's controls, banking app approvals, any 2FA prompt that isn't a
TOTP code you can read elsewhere, QR-scanner-based check-ins, anything that
expects you to switch apps to confirm something.

This is a meaningfully smaller loss than the lockdown first appears to impose.
Worth stating plainly in the README, because "I'd lose my car key" is the kind of
objection that kills the idea before anyone checks whether it's true.

### Wallet splits three ways

I'd been treating Wallet as one thing that "still works." It isn't, and the three
cases have very different confidence levels.

**1. Express Mode — works, and this one is now confirmed rather than reasoned.**
Transit cards, car keys, home and hotel keys need no authentication and no
interaction. Apple's platform security documentation is explicit that the NFC
controller performs Express Card transactions *independently of iOS* — they work
in power reserve after the battery is flat, "under the same conditions as when
iOS is running." If it works when iOS effectively isn't, a UI-layer restriction
like Guided Access cannot touch it. The
[screen-not-radios principle](#what-the-lockdown-does-not-touch) at its
clearest.

**2. Authenticated Apple Pay — blocked by default, possibly recoverable.**
Paying at a retail terminal means double-clicking the side button. But
[the side button is disabled by default](#2-the-side-button-is-disabled-by-default)
during a Guided Access session, so with stock options the double-click never
reaches Wallet. Users hitting this report having to choose between Guided Access
and Apple Pay.

Enabling Sleep/Wake in Options makes the button live again, which *should*
restore the double-click — but "the button works" and "double-click summons
Wallet over a pinned app" are different claims and only the first is confirmed.
[Row 15](notifications.md#verification-matrix) now tests it with the button
explicitly enabled. Apple's own suggested workaround for the conflict is
**AssistiveTouch**, which can trigger the payment confirmation without the
physical double-click; worth testing as a fallback.

If it turns out to be blocked, the fallbacks are decent:

- **An Apple Watch pays independently** of the phone's lock state. This keeps
  coming up — the Watch also potentially solves the
  [notification glance](notifications.md#background-services-matrix), and it's
  starting to look less like an accessory and more like the natural complement
  to a locked-down phone.
- Exit Guided Access to pay. Triple-click and Face ID is a couple of seconds,
  just an ugly couple of seconds at a till.
- Carry a card.

**3. Anything with a barcode — lost.** Boarding passes, event tickets, loyalty
cards, ID. These need the Wallet app's *screen*, and the lockdown denies other
apps' UI by definition. No workaround short of exiting Guided Access, which for a
boarding pass at a gate is genuinely the right move rather than a failure.

This is a real gap and belongs in the same honest list as telephony. It's mild —
exiting takes seconds and air travel already involves a dozen worse indignities —
but the docs shouldn't claim Wallet "works" when a third of it doesn't.

### Configure before you lock

A recurring pattern across this project, now explicit. These are one-time
foreground setup steps that must all happen before the device is pinned:

- Mail and calendar accounts in Settings — see
  [calendar](features/calendar.md#approach)
- Tesla phone key pairing (app + key card)
- Wallet cards and keys, with Express Mode enabled
- Apple Watch and AirPods pairing
- Wifi networks, including any you'll need away from home
- **Mirror Display Auto-Lock** in Accessibility → Guided Access, and **Sleep/Wake
  Button** in the session Options — see
  [session options you must change](#session-options-you-must-change). Both are
  off by default and both matter.
- **The VPN to the sandbox** — Tailscale or WireGuard, installed, signed in, and
  set to connect on demand. `sanctumd` is loopback-only, so without this the
  Talk tab can never reach it from cellular or any foreign network, and the
  VPN's own app is unreachable once the lockdown is on. Verify it reconnects
  after a reboot before going further.
- Sanctum's own [sandbox enrollment](features/claude.md#auth) and
  [messaging roster](features/messaging.md#people-not-accounts)
- Signal and WhatsApp device linking, which requires the official apps

Plus the power configuration pass — Low Power Mode, Background App Refresh,
5G/LTE, Raise to Wake and the rest — which is listed in
[power](power.md#what-only-pre-lockdown-configuration-can-do). None of it is
reachable from an app, so it all has to happen here.

Order matters too: `sanctumd` has to be running and reachable *before* the phone
is enrolled, because enrollment consumes a live one-time code. Sandbox first,
app second, device configuration third, lockdown last.

M0 should produce this as an actual printed checklist, not a paragraph. Getting
locked into a phone that can't join a wifi network is a bad afternoon.

## What the lockdown takes away

Things Sanctum must either replace or consciously live without:

| Lost | Consequence for Sanctum |
| --- | --- |
| Other apps' notifications | Sanctum is the only thing that can get your attention. Our notification discipline has to be near-perfect. |
| Phone / Messages | **Open question.** See [roadmap](roadmap.md#open-questions) — no third-party app can place a normal call, and SMS cannot be read programmatically. |
| Camera | Only reachable via an in-app picker; no system camera. Photo capture is a feature we'd have to build. |
| Browser | `SFSafariViewController` still works *inside* Guided Access. This is our one seam to the wider web — deliberately narrow (see below). |
| Maps / navigation | Not covered. A known gap. |
| Wallet passes with barcodes — boarding passes, tickets, loyalty, ID | **Lost.** They need the Wallet app's screen, which the lockdown denies. See [Wallet](#wallet-splits-three-ways). |

### The browser seam

Links in mail and in Claude's replies have to go somewhere. `SFSafariViewController`
presents in-app and returns when dismissed, so it does not break the lockdown —
but it *is* a full browser with a URL bar, and it is the obvious hole in the
whole design.

Policy (v1): links open in `SFSafariViewController`, with an allowlist toggle in
settings. When the allowlist is on, off-list links show a "blocked, copy link"
sheet instead. Default: allowlist off, because a half-working mail client is
worse than a seam.

## Emergency access

Non-negotiable: a phone locked into one app must never be a phone that can't call
for help.

- The system **Emergency SOS** gesture (hold side + volume) works during Guided
  Access and is not something Sanctum can or should suppress.
- Guided Access can be exited with the passcode or Face ID at any time.
- Sanctum's own lock screen (if we add one) must never gate the app behind a
  server check. Local-only auth.

Document these in the app's own onboarding, not just here.

## UI consequences

- **Home indicator hidden.** The bottom safe-area inset changes. Don't hardcode.
- **Side-button triple-click is reserved.** Never bind a gesture that trains the
  user to fight it.
- **The screen sleeps — but only once you've enabled Mirror Display Auto-Lock.**
  Guided Access ignores the system Auto-Lock by default. Sanctum itself draws no
  always-on clock and never disables the idle timer, but the app cannot rescue a
  misconfigured session; see
  [session options](#session-options-you-must-change). Dark surfaces remain the
  default after dark, for the 3am glance rather than for a display left lit.
- **Long-lived, but not lit.** The session may last weeks even though the screen
  doesn't, so leaks, unbounded caches and drifting timers are still real bugs
  rather than theoretical ones. Long-uptime soak testing stays in CI.
