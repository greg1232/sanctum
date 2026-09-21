# Calendar

The easiest of the four, and the one that sets the quality bar.

## Approach

Use **EventKit** against the device's own calendar database. Accounts (iCloud,
Google, Exchange, CalDAV) are configured once in Settings during phone setup,
before the lockdown; Sanctum then reads and writes the local store and iOS
handles sync. We implement no calendar protocol ourselves.

Request `EKEventStore` full access (`NSCalendarsFullAccessUsageDescription`).
Write-only access is not enough — the whole point is seeing the day.

## Surfaces

1. **Today** (default). A vertical timeline of the current day, now-line pinned,
   all-day events in a header strip. This is the screen someone glances at.
2. **Next 7 days.** An agenda list, grouped by day, collapsing empty days to a
   single row. Not a month grid — a month grid is a map, and we want a schedule.
3. **Event detail.** Title, time, location, notes, attendees, conference link.
   Edit in place.
4. **Quick add.** One text field, natural-language parsed ("lunch with sam
   thursday 12:30"). Parsing happens on-device; if it's ambiguous, show the
   structured form pre-filled rather than guessing.

Explicitly not in v1: month grid, multi-calendar color management UI, invitation
RSVP triage, availability/free-busy search.

## Behavior details

- **Now-line and "starts in N minutes"** are the point of the Today screen.
  Time-to-next-event should be legible from across a room.
- **Launch routing:** if an event starts within 15 minutes, Sanctum opens on
  Calendar regardless of last tab (see [architecture](../architecture.md#navigation-model)).
- **Notifications:** one alert per event, at the event's own alarm settings.
  Sanctum does not add its own default reminders — respect what the calendar says.
- **Conference links** (Meet/Zoom/Teams) are detected in location + notes and
  surfaced as a single button. Under Guided Access that button opens
  `SFSafariViewController`; joining a real video call is out of scope for v1 and
  should be *said* to be out of scope in the UI rather than half-working.
- **Travel time / location awareness:** deferred. Requires location permission
  and earns its keep only with commute data we don't have yet.

## Testing

EventKit is awkward under test. Wrap it in a `CalendarStore` protocol in
`SanctumKit`; the real implementation is thin enough to eyeball, and every view
model is tested against an in-memory fake with an injected `Clock`. Timezone and
DST transitions get explicit test cases — a nightstand alarm clock app that gets
DST wrong is worthless.
