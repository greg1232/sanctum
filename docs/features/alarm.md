# Alarm

A phone locked into one app on a nightstand is an alarm clock. This has to be
boringly reliable — more reliable than anything else in Sanctum, because the
failure mode is "you overslept."

## The iOS constraint

Historically, third-party apps could not schedule an alarm that would wake a
silenced or sleeping device; only Apple's Clock app could. `UNNotification`
sounds respect the ringer switch and Focus modes.

Two routes out:

1. **AlarmKit** (iOS 26+) — the supported API for real alarms from third-party
   apps: rings through silent mode and Focus, presents a full-screen alert with
   snooze/stop. Requires the alarm entitlement and a clear user-facing purpose.
   **This is the primary implementation.**
2. **Foreground audio fallback** — because Sanctum is *already the foreground app
   on a phone that never sleeps*, it can simply play audio at the right time with
   an active `AVAudioSession`. This is normally a fragile hack; under Guided
   Access with auto-lock disabled it is nearly the happy path.

Ship both. AlarmKit is correct; the foreground path is the belt to its
suspenders, and it costs almost nothing given the deployment model.

Never rely on `UNTimeIntervalNotificationTrigger` alone. It is not an alarm.

## Surfaces

1. **Clock face.** Large, legible across a dark room. True-black background,
   dimmed amber/red at night, brightness following ambient light. This is the
   screen the phone sits on all night, so it must emit as little light as
   possible while staying readable.
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
can't: at bedtime, drop to the clock face and refuse to show mail or chat until
the morning alarm. That's a real feature of the containment model, and a good
candidate for the first post-v1 addition — see [roadmap](../roadmap.md).
