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

Target: the clock face redraws **once per minute** while visible and is otherwise
completely static. Everything else in the app draws on interaction only. This
matters less now that no screen stays lit unattended, but a static screen is
still free and an animated one isn't.

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

## Sleep and wake — the screen sleeps, always

Earlier drafts argued for keeping the screen on with an app-drawn near-black idle
face, on the grounds that letting it sleep would force us into push
infrastructure we didn't want. A later draft softened that to a charging-only
"dock mode."

**Both are gone. The screen sleeps like any phone's, plugged in or not** — with
one setup caveat that is not optional: a Guided Access session ignores your
Auto-Lock setting unless
[Mirror Display Auto-Lock is enabled](guided-access.md#1-screen-sleep-behaves-differently--and-the-default-is-bad),
falling back to a 20-minute timeout — and with the side button disabled by
default you can't lock it manually either. Every number in this document assumes
both are configured.
 Sanctum
has no always-on clock, no persistent display, and no `isIdleTimerDisabled`
anywhere in the codebase. If you want a glanceable clock on a nightstand, that's
the system lock screen's job and a Settings toggle you own — not a surface we
build and not a mode we manage.

What this buys beyond battery:

- **No dual-mode UI.** The clock face is a screen you open, like every other
  screen. There is no second rendering path with different rules.
- **Fewer long-uptime bugs.** The app now backgrounds and suspends regularly
  instead of running lit for weeks, which retires a whole class of leak, drift
  and unbounded-cache failures.
- **One less thing to get wrong on a device you can't easily debug.**

### What sleeping actually costs

When the screen locks, Sanctum backgrounds and is suspended within seconds. The
WebSocket to [`sanctumd`](features/claude.md) dies. Timely messages therefore
require **APNs**. There is no way around this: `audio` background mode (playing
silence to stay alive) is a hack with a real power cost of its own; `voip` now
requires PushKit pushes that must produce a CallKit call; `BGAppRefreshTask` is
opportunistic and nowhere near timely enough.

### The correction: no relay is needed

The earlier framing — that push "tempts us toward hosted infrastructure we said
we wouldn't run" — was overstated. Modern APNs uses token-based auth: an HTTP/2
request to Apple, signed with a JWT from a `.p8` key issued by your developer
account. **`sanctumd` can be the push provider directly.** The device registers
for remote notifications, hands its token to `sanctumd` over the socket it
already has, and `sanctumd` posts to Apple when something arrives.

That's a p8 file and an HTTP/2 client on a box you already own. Apple's APNs sits
in the path — but it sits in the path of every notification on the device
regardless, so it costs us nothing we hadn't already conceded.

The real prerequisite is the **$99/yr Apple Developer Program**, since push
requires a paid account. That's almost certainly already required for AlarmKit's
entitlement, Time Sensitive notifications, and a signing certificate that lasts a
year instead of seven days.

### Keep content out of the payload

Push payloads pass through Apple. For a messaging app that matters, so:

- Alert pushes carry **minimal text** — "Sam sent a message", never the body.
- The app fetches real content over its own socket on wake, and may raise a
  *local* notification with detail once it has it.
- Silent (`content-available`) pushes are the privacy-maximal version but are
  throttled and not guaranteed, so they can't be the only mechanism for anything
  time-sensitive. Use a minimal alert push, and treat silent pushes as an
  optimization.
- Roster messages should use `interruption-level: time-sensitive` so they break
  through Focus.

### What still works with no push at all

Worth being precise, because it bounds the blast radius:

- **Alarms and timers:** unaffected. AlarmKit is local and fires from a sleeping
  device by design. Nothing about push touches [alarm](features/alarm.md).
- **Calendar alerts:** unaffected. Local notifications, scheduled on device.
- **Messages and Claude:** these are the only surfaces that degrade, which
  matches the dependency table in [architecture](architecture.md#offline--online-split).

A sandbox outage still costs you nothing you'd oversleep over.

### One place the lockdown helps

Under Guided Access there is no app switcher, so the user cannot force-quit
Sanctum. That matters: a user-terminated app stops receiving silent pushes
entirely, while an app iOS itself evicted can still be relaunched by one. The
lockdown makes the background story *more* reliable, not less.

## Maximizing battery life

### The largest win is already built in

Before any settings discussion: on a normal phone, battery goes to **screen-on
time**, and screen-on time goes to apps engineered to hold you there. Sanctum has
none. Every list in it is finite, nothing scrolls forever, and there is no app to
switch to when you're bored.

A phone whose entire software surface is four bounded tools has a fraction of the
screen-on time of a normal phone, and screen-on time is the dominant term. **The
feature set is the battery optimization.** Everything below is the small term —
worth doing, but don't mistake it for the main event.

### What Sanctum can control directly

The app's own behavior, no permissions required:

- **`UIScreen.main.brightness`** — settable. The app can drive brightness down
  for its own dark surfaces and the system restores it on exit. Combined with
  OLED black, this is the biggest lever we actually hold.
- **`isIdleTimerDisabled`** — whether the screen sleeps. Per the dock-mode
  decision above: `false` normally, `true` only while charging.
- **Refresh rate.** `CADisplayLink.preferredFrameRateRange` requests a low frame
  rate on ProMotion displays. A static clock should ask for the floor.
- **Network batching and keepalive tuning** — covered above; the radio tail is
  ours to manage.
- **`ProcessInfo.isLowPowerModeEnabled`** — readable, not settable. React to it:
  stretch poll intervals, stop non-essential sync.
- **`ProcessInfo.thermalState`** — back off under thermal pressure rather than
  competing with the system for a throttled SoC.
- **`UIApplication.backgroundRefreshStatus`** and `NWPathMonitor` (`isExpensive`
  for cellular, `isConstrained` for Low Data Mode) — readable, so the app can at
  least *audit* the environment it's running in.

### What only pre-lockdown configuration can do

There is no API for any of this. It belongs in the
[configure before you lock](guided-access.md#configure-before-you-lock)
checklist, and most of it costs nothing precisely *because* no other app is in
use:

| Setting | Why it's free here |
| --- | --- |
| **Low Power Mode on** | Throttles background activity and caps ProMotion at 60Hz. On a device doing almost nothing, imperceptible. ⚠️ Auto-disables at 80% charge — see below. |
| **Background App Refresh off (global)** | Nothing else needs to refresh. Does not affect BLE background modes, so the [car key still works](guided-access.md#worked-example-a-tesla-phone-key). |
| **LTE instead of 5G**, or cellular data off entirely | 5G costs real power. If this is a wifi-first device, the whole radio can go. |
| **Wi-Fi Assist off** | Stops silent, expensive failover to cellular. |
| **Raise to Wake and Tap to Wake off** | A phone in a pocket lights its screen dozens of times a day for nobody. |
| **System always-on display** | Your call. Sanctum draws no clock of its own, so if you want a nightstand glance this is where it comes from — Apple's 1Hz implementation, not ours. Off is cheaper. |
| **Dark Mode forced** | OLED again, and it matches the design anyway. |
| **Reduce Motion / Reduce Transparency** | Small GPU savings, consistent with the aesthetic. |
| **Automatic app updates and downloads off** | There is no App Store use. |
| **System Mail fetch off** for accounts Sanctum syncs | Otherwise the same mailbox is fetched twice by two clients. |
| **"Hey Siri" off** | Small always-listening cost — but see the Siri question below before assuming this is purely a win. |

Two that need thought rather than a blanket toggle:

- **Bluetooth: leave it on.** Car key, Watch, AirPods all depend on it, and these
  are exactly the things the lockdown otherwise preserves.
- **Location Services: don't disable globally.** It would take Find My with it.
  Turn it off per-app instead; Sanctum requests none.

### Supervision buys more than the lock

If the device is supervised for
[Autonomous Single App Mode](guided-access.md#b--autonomous-single-app-mode-asam),
MDM restriction payloads can *enforce* much of the table above declaratively
rather than relying on someone remembering to flip switches — App Store off, Siri
off, AirDrop off, iCloud services off, and more.

That strengthens the case for supervision. The tax isn't only buying a better
lock; it's buying a reproducible device configuration.

### What we simply cannot do

Be clear about the boundary so nobody plans around a fantasy: an app cannot
toggle cellular, Bluetooth, 5G, Low Power Mode, Background App Refresh, or
Location Services. It cannot deep-link to those Settings panes either — the
private URL schemes that once allowed it are blocked, and
`openSettingsURLString` only reaches Sanctum's own page.

What Sanctum *can* do is **audit and nag once**: check what it can read
(background refresh status, Low Power Mode, cellular vs wifi, battery state), and
show a one-time setup review listing what's still unset. Not a recurring warning
— a checklist that goes away when you've done it.

### Questions to settle

- **Does Low Power Mode survive the nightstand?** It auto-disables at 80% charge,
  which is exactly what a docked phone hits every night. A Shortcuts personal
  automation triggering on charger disconnect could re-enable it — *if* Shortcuts
  automations fire during a Guided Access session. Unverified, and worth knowing,
  because it also tells us whether Shortcuts is available as a general escape
  valve.
- **Does Siri work under Guided Access?** If it does, it's simultaneously a
  lockdown hole and a genuine capability — voice is a good input method for a
  phone with a deliberately small UI. Decide which before disabling it.
- **Cellular at all?** Still the open question from below. It's the single
  largest configuration lever and it depends entirely on whether this phone
  leaves the house.

## How long would it actually last?

⚠️ **These are modeled estimates, not measurements.** They're built from published
battery capacities and Apple's rated runtimes, and the component figures below
could each be off by half. They're here to check whether the design decisions
make sense, not to promise a number. The M0 soak test replaces all of it.

Baseline: a standard iPhone is roughly **13 Wh** (a Pro Max is closer to 17–18;
scale accordingly). Battery health scales everything linearly — an 85% battery
gives 85% of these figures.

### Component draw

| State | Estimated draw | As % of a 13 Wh battery |
| --- | --- | --- |
| Deep standby — wifi, Low Power Mode, no background refresh | ~50 mW | **~0.4 %/hr** |
| Deep standby — cellular, poor signal | up to ~200 mW | ~1.5 %/hr |
| Screen on, true black, minimum brightness, static | ~400 mW | **~3 %/hr** |
| Active use — reading mail or a thread, dark UI, network | ~2 W | **~15 %/hr** |
| One APNs push wake | ~1 J | ~0.002% — **negligible** |

### Scenarios

| Scenario | Daily drain | Expected life |
| --- | --- | --- |
| **Carried, wifi + cellular, ~1 hr/day screen-on** | 23 hr standby (9%) + 1 hr active (15%) ≈ 25% | **~4 days** |
| **Carried, wifi only, ~20 min/day screen-on** | 6% + 5% ≈ 11% | **~9 days** |
| **Heavy day** — poor cellular, lots of Claude streaming | ~60–80% | ~1.5 days |
| **Overnight, asleep on a charger** | n/a | indefinite; see [the plugged-in problem](#the-plugged-in-problem) |

For calibration: run the same model on a normal phone — 4 hours of screen-on time
and heavier background activity gives ~75%/day, i.e. charging every night. That
matches reality, which suggests the model isn't wildly off.

### What the numbers actually tell us

**1. Multi-day battery life is the realistic target, and it's not from
optimization.** It's from screen-on time. A normal phone spends 3–5 hours a day
lit because several apps are engineered to make that happen. Sanctum has no such
app, so the dominant term collapses — and everything else follows from that one
fact rather than from anything clever.

**2. Push costs essentially nothing.** At ~0.002% per wake, even a hundred
notifications a day is a rounding error. The
[decision to let the screen sleep](#sleep-and-wake--yes-avoid-the-always-on-display)
gave up nothing measurable and bought multi-day life. That was the right call and
this is the arithmetic confirming it.

**3. An always-on clock was never worth it.** Leaving the screen lit overnight
costs ~24% — a quarter of the battery to display a clock nobody looks at for
seven of those eight hours. Removing it outright is the difference between a
four-day phone and a two-day one, and it removes a mode rather than adding one.

**4. Cellular signal is the largest variable you don't control.** Standby in poor
signal is roughly 4× standby on wifi, which alone moves a four-day phone to
two-and-a-half. This is the strongest argument for the wifi-only configuration
where it's viable, and it's why "does this phone leave the house" keeps being the
question everything hinges on.

**5. One hour of active use costs more than a full day of standby.** Fifteen
percent versus nine. Which means no optimization in this document matters as much
as the app not being interesting to stare at — and that's a design property, not
an engineering one.

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

**States to budget, cheapest to most expensive** — the estimates above are the
hypothesis each of these is testing:

| State | Target |
| --- | --- |
| Asleep, wifi, occasional push | the floor — measure and set the target from it |
| Clock or calendar visible, idle, dark | measure; it should be dominated by brightness |
| Active reading (mail, threads) | unbudgeted, it's bounded by attention |
| Streaming a Claude response | bounded by the turn |
| Reconnect storm | **should be impossible** — backoff must be capped and tested |

The last row is the only one that's a bug rather than a cost. Everything else is
a tradeoff; an uncapped retry loop is just a mistake that flattens the battery.

## Open questions

- **Cellular at all?** If this phone lives on wifi, disabling cellular data is
  free battery. If it's also the phone that needs to work away from home, no.
  Related to the telephony question in [roadmap](roadmap.md#open-questions).
- **Keepalive interval**, empirically: the longest ping that survives typical
  home-router and carrier NAT timeouts without dropping the socket.
- **Should Sanctum manage its own brightness?** Ambient-light-driven dimming is
  obviously right for a nightstand, but it means the app fights the user's manual
  brightness setting. Probably: adjust automatically, yield permanently to any
  manual change until the next day boundary.
