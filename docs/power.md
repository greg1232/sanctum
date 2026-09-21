# Power

iOS is tuned for apps that are briefly foreground and otherwise suspended.
Sanctum is foreground continuously, potentially for weeks, often with the screen
lit. Every default assumption about power is wrong for us, in both directions:
some things matter far more than usual, and some things stop mattering at all.

There are also two distinct problems, and the second one is the one people miss.

- **On battery:** what drains it.
- **Plugged in:** what *degrades* it. A phone held at 100% and slightly warm
  24/7 is the documented failure mode for kiosk devices — swollen batteries in
  wall-mounted iPads are not a rare story. On a nightstand, this is the real
  power problem, and no amount of efficient code addresses it.

## What burns power, ranked

### 1. The display — dominant, and it's a design lever

On an OLED iPhone, screen power scales with **what you draw**, not just how long.
Lit pixels cost; black pixels cost close to nothing. A full-white mail view and a
true-black clock face at the same brightness are not remotely the same amount of
energy. Brightness is the other multiplier, and it's a large one.

This means the biggest power decision in the app is a *visual design* decision,
and it was already made for legibility reasons in [alarm](features/alarm.md):
true-black background, dim amber digits, minimum readable brightness. That choice
is worth more than every optimization below combined.

### 2. Redraw rate — the silent one

A screen that renders at 60 or 120Hz forever never lets the display pipeline or
GPU idle. The offenders are subtle and all of them are aesthetic temptations:

- A sweeping second hand, or **a blinking colon**.
- Any looping Core Animation, spinner, or "breathing" effect.
- A clock showing **seconds** at all — it forces 60× the redraws of one showing
  only hours and minutes, to tell you something you do not need at 3am.

Target: the night clock face should redraw **once per minute** and be otherwise
completely static. Everything else in the app draws on interaction only.

### 3. Radio — and specifically the tail

Cellular power scales with how hard the radio has to shout; poor signal is far
worse than good, and 5G worse than LTE. Prefer wifi; consider whether this device
needs cellular data at all.

More importantly: **every radio wake has a tail.** The modem stays in a
high-power state for seconds after a transfer completes, so ten small transfers
spread across a minute cost dramatically more than the same bytes sent at once.

Direct consequences for our design:

- **Keepalives are the whole ballgame.** Our persistent WebSocket to
  [`sanctumd`](features/claude.md) is cheaper than polling only if its ping
  interval is long. A 10-second keepalive holds the radio warm permanently and
  would be worse than polling. Tension: too long and NAT drops the connection,
  producing reconnect storms that cost more than the pings saved. **Prefer long
  keepalives plus fast, cheap reconnect** over short keepalives.
- **Align everything to one schedule.** Mail refresh, Claude keepalive, message
  keepalive, and any token refresh should wake the radio *together*, not on three
  independent timers. This is a `sanctumd` protocol concern as much as a client
  one — one socket carrying Claude and messages already helps a lot.
- IMAP IDLE's natural ~29-minute re-IDLE cycle is a reasonable anchor interval.

### 4. Timer coalescing

Many small timers prevent the SoC from reaching deep idle. Five features each
running their own one-second tick is meaningfully worse than one heartbeat
driving all periodic work. Set `tolerance` on every timer so iOS is free to
coalesce, and never use a repeating timer where a recomputed next-fire-date works
— which [alarm](features/alarm.md) already requires for correctness reasons.

### 5. Location — free win by omission

GPS is expensive and we use none. Calendar deliberately defers travel-time
features. Keeping location out of the app is worth real battery, and it's worth
noticing that the privacy-motivated choice and the power-motivated choice are the
same choice.

### 6. Thermals, as an amplifier

Screen on + charging + sustained CPU produces heat; heat causes throttling *and*
accelerates battery aging. A phone in a thick case, charging all night, running a
lit display, is warm for eight hours straight. This compounds the plugged-in
problem below.

### 7. Memory — not power, but adjacent

A process running for weeks that leaks will eventually be jetsam'd. That costs a
relaunch, and per the
[notification matrix](notifications.md#verification-matrix) row 12 it may also
silently end the lockdown. Long-uptime leak testing is a power *and* correctness
requirement.

## The plugged-in problem

If the phone lives on a charger, battery *life* stops mattering and battery
*health* becomes the whole story. Sitting at 100% state-of-charge at elevated
temperature is the worst case for lithium-ion aging.

Mitigations, roughly in order of effectiveness:

- **Charge limit.** Recent iPhones offer an 80% cap in Settings. For a device
  that never leaves the nightstand, this costs nothing and is the single best
  intervention.
- **Optimized Battery Charging**, if a hard cap isn't available.
- **A smart plug on a schedule**, cycling the charger rather than holding it.
- **Take the case off.** Genuinely material for a device that's warm 8h/night.

Sanctum should surface this: if the app can see it's been on the charger above
~95% for a sustained period, say so once, in settings, and link to the setting.
Not a nag — a phone this deliberate deserves to tell you when its own deployment
model is quietly destroying it.

## The auto-lock question, revisited

[Notifications](notifications.md#the-reframe-sanctum-is-foreground-so-it-mostly-doesnt-need-notifications)
framed auto-lock as a binary with a nasty tradeoff: screen always on means no
push relay needed but heavy drain; auto-lock on means far less power but an APNs
relay we don't want to run.

Power suggests a third option that may be the actual answer:

> **Keep the screen on, but let Sanctum draw its own idle state** — a
> near-entirely-black face at minimum brightness, redrawing once a minute.

You keep glanceability, you keep the open socket, you need no push
infrastructure, and you avoid most of the display cost, because on OLED the cost
*is* the lit pixels. It is an app-drawn approximation of always-on display, and
it fits a device that is plugged in anyway.

This is now my preferred answer to the auto-lock question. It needs measuring
before it's a decision.

## Budget and measurement

Don't guess at any of this — the ranking above is sound, but magnitudes on real
hardware are the only thing that should drive optimization work.

**Instrument with:**

- **MetricKit** — the right tool for a long-running app. Daily payloads include
  CPU, display, animation, and location metrics, including average pixel
  luminance. Wire it up in M0 and let real usage report itself.
- **Xcode Energy gauge / Instruments Energy Log** for attributing spikes.
- **A soak test**: leave the device unplugged in each UI state for an hour and
  record percent-per-hour.

**States to budget, cheapest to most expensive:**

| State | Target |
| --- | --- |
| Night face, min brightness, socket idle | the floor — measure and set the target from it |
| Day clock / calendar idle | ≤ 2× the night face |
| Active reading (mail, threads) | unbudgeted, it's bounded by attention |
| Streaming a Claude response | bounded by the turn |
| Reconnect storm | **should be impossible** — backoff must be capped and tested |

The last row is the only one that's a bug rather than a cost. Everything else is
a tradeoff; an uncapped retry loop is just a mistake that flattens the battery.

## Open questions

- **Does the dim-idle face actually save what I think it does?** Measure a
  true-black minimum-brightness static face against screen-off. If the gap is
  small, the auto-lock question resolves itself and we never build a push relay.
- **Cellular at all?** If this phone lives on wifi, disabling cellular data is
  free battery. If it's also the phone that needs to work away from home, no.
  Related to the telephony question in [roadmap](roadmap.md#open-questions).
- **Keepalive interval**, empirically: the longest ping that survives typical
  home-router and carrier NAT timeouts without dropping the socket.
- **Should Sanctum manage its own brightness?** Ambient-light-driven dimming is
  obviously right for a nightstand, but it means the app fights the user's manual
  brightness setting. Probably: adjust automatically, yield permanently to any
  manual change until the next day boundary.
