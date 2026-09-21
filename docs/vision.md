# Vision

## The problem

A modern smartphone is a general-purpose attention machine that happens to also
hold your calendar. Every useful thing on it is three taps from an infinite feed.
The usual remedies — screen time limits, app deletion, grayscale — all fail the
same way: the escape hatch is always one gesture away, and the device is designed
to make that gesture easy.

## The move

Stop negotiating with the home screen. Remove it.

iOS has a supported mechanism for this: **Guided Access** (and its managed
sibling, Single App Mode). The phone is pinned to one app and cannot leave
without a passcode. Sanctum is built to be that one app.

This inverts the design problem. Normally an app competes for attention against
everything else on the device. Sanctum has no competition — and therefore no
excuse. If something is genuinely necessary, Sanctum must provide it. If Sanctum
doesn't provide it, you genuinely don't do it on this phone.

## Principles

1. **The phone is a tool, not a venue.** Every screen answers a question or
   completes a task, then gets out of the way. Nothing is endless.
2. **No feeds, no badges, no infinite scroll.** Lists are finite and bounded by
   real-world quantities (today's events, unread mail, one conversation).
3. **Containment over blocking.** We don't fight the user with willpower prompts.
   The lockdown is structural, decided once, at setup.
4. **The escape hatch is real but deliberate.** Guided Access is exited with a
   passcode. Sanctum never pretends to be inescapable — that would be both a lie
   and, in an emergency, dangerous. See [Escape hatches](guided-access.md#escape-hatches).
5. **Degrade honestly.** No network, no sandbox, expired token — the app says so
   plainly and keeps the offline surfaces working. Calendar and alarms must never
   depend on the backend.
6. **Your data stays yours.** Mail and calendar sync directly from your providers
   to the device. Nothing routes through a Sanctum-operated server. The only
   backend is *your own* sandbox.

## Non-goals

- Not a launcher or a home-screen replacement. Sanctum does not try to host other
  apps; on iOS it couldn't anyway.
- Not a parental-control product. This is a device you choose to constrain for
  yourself.
- Not a social app. There is no Sanctum account, no directory, no presence.
- Not an offline-first email archive. Mail sync is recent-window, not "all mail".

## The test

> Can I hand someone a phone running only Sanctum for a week, and have them lose
> nothing they actually needed?

If the answer is no, the missing thing is either a feature we owe them or a habit
the phone was manufacturing. Telling those two apart is the whole design job.
