# Guided Access

Sanctum's premise is that the phone cannot leave the app. iOS offers two ways to
do that, and they are not equivalent.

## Option A — Guided Access (no MDM)

Accessibility → Guided Access. Triple-click the side button inside Sanctum to
start a session; triple-click and enter a passcode (or Face ID) to end it.

- **Pros:** zero infrastructure, works on any phone in about 90 seconds, easy to
  exit in an emergency.
- **Cons:** it's a *session*, not a device state. It does not survive a reboot —
  the phone comes up on the home screen and someone has to re-enter Guided
  Access manually. Also disables the Home indicator and some system gestures in
  ways that affect our UI (see below).

This is the default setup and what the docs assume.

## Option B — Single App Mode (supervised + MDM)

Supervise the phone with Apple Configurator, enroll it, and push a Single App
Mode payload locking it to Sanctum's bundle ID.

- **Pros:** survives reboot, survives update, genuinely persistent. The phone is
  a Sanctum appliance.
- **Cons:** requires supervising the device (a full erase), a Mac, and an MDM or
  Configurator workflow. Exiting means un-enrolling.

Recommended for a dedicated second phone. Overkill for "my main phone, evenings
only."

## What the lockdown takes away

Things Sanctum must either replace or consciously live without:

| Lost | Consequence for Sanctum |
| --- | --- |
| Other apps' notifications | Sanctum is the only thing that can get your attention. Our notification discipline has to be near-perfect. |
| Phone / Messages | **Open question.** See [roadmap](roadmap.md#open-questions) — no third-party app can place a normal call, and SMS cannot be read programmatically. |
| Camera | Only reachable via an in-app picker; no system camera. Photo capture is a feature we'd have to build. |
| Browser | `SFSafariViewController` still works *inside* Guided Access. This is our one seam to the wider web — deliberately narrow (see below). |
| Maps / navigation | Not covered. A known gap. |
| Wallet / Apple Pay | Double-click side button still works; Guided Access does not block it. |

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
- **Screen may never sleep.** A Guided Access phone on a nightstand is often set
  to never auto-lock. Assume the UI is *always visible*: true-black night mode,
  no bright surfaces after dark, no animation that loops forever.
- **Single session, long-lived.** The app may stay foregrounded for days. Leaks,
  unbounded caches, and timers that drift all become real bugs rather than
  theoretical ones. Long-uptime soak testing is part of CI.
