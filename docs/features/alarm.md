# Alarm

A phone locked into one app on a nightstand is an alarm clock. This has to be
boringly reliable — more reliable than anything else in Sanctum, because the
failure mode is "you overslept."

## The iOS constraint

Historically, third-party apps could not schedule an alarm that would wake a
silenced or sleeping device; only Apple's Clock app could. `UNNotification`
sounds respect the ringer switch and Focus modes.

**AlarmKit** (iOS 26+) is the answer: the supported API for real alarms from
third-party apps, ringing through silent mode and Focus with a full-screen alert
carrying snooze and stop. It needs the alarm entitlement and a clear user-facing
purpose. It is local, so it fires from a sleeping phone with no network and no
sandbox.

Never rely on `UNTimeIntervalNotificationTrigger` alone. It is not an alarm.

### There is no longer a second mechanism

An earlier draft paired AlarmKit with a foreground audio fallback, on the
reasoning that Sanctum was already the foreground app on a phone that never
slept. [That premise is gone](../power.md#sleep-and-wake--the-screen-sleeps-always)
— the screen sleeps, the app suspends, and a suspended app cannot start playing
audio at a scheduled time. Keeping it alive with a silent audio session is
exactly the power-wasting hack we rejected elsewhere.

So **AlarmKit is the sole mechanism**, and the redundancy is gone from a feature
whose whole requirement was being boringly reliable. Two consequences:

1. **Verify it properly.** [Row 5](../notifications.md#verification-matrix) of
   the matrix — fires through silent mode and Focus, from a sleeping device, on
   schedule, repeatedly, across a DST boundary — is not a nice-to-have. It is the
   feature.
2. **Recommend a system backstop.** A recurring alarm in Apple's Clock app,
   set before lockdown, costs nothing and is independent of our code entirely.
   It can't be changed later without exiting Guided Access, so it suits a fixed
   weekday wake time rather than anything you adjust. [Row 4](../notifications.md#verification-matrix)
   asks whether Clock alarms fire during a session; if they do, say so in
   onboarding and let people decide.

## Surfaces

1. **Clock face.** A screen you open, not one that stays on — Sanctum has no
   always-on display. Still true-black with dimmed amber/red at night and
   brightness following ambient light, because the case that matters is checking
   the time at 3am without being blinded.
2. **Alarms list.** Repeating schedules, labels, per-alarm sound, on/off toggles.
3. **Ringing.** Full-screen, unmissable, Stop and Snooze as large targets. No
   other UI reachable until dismissed.
4. **Timers.** One-shot countdowns. Multiple concurrent timers, named.

## Behavior details

- **Launch routing:** a ringing alarm takes over the app regardless of tab.
- **DST and timezone.** "7:00 every weekday" means local wall-clock 7:00, across
  DST transitions and travel. Explicit test matrix; this is the classic bug.
- **Long uptime.** The app may run for weeks. Do not schedule with a single
  long-lived `Timer`; re-derive the next fire date from the wall clock on every
  foreground, significant-time-change notification, and periodic tick.
- **Volume ramp.** Start quiet, escalate over ~30 seconds. Stop is always
  instant and always one tap.
- **Snooze** default 9 minutes, configurable, with a hard cap on consecutive
  snoozes that is shown, not enforced silently.

## Bedtime hooks (later)

Because Sanctum owns the whole device, it can do what a standalone alarm app
can't: at bedtime, collapse to the clock face and refuse to show mail or chat
until the morning alarm. That's a real feature of the containment model, and a good
candidate for the first post-v1 addition — see [roadmap](../roadmap.md).
