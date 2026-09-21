# Integrations

How Sanctum talks to things that aren't calendar, mail, messages or alarms —
cars, home automation, anything else with an API. Tesla is the worked example
because it came up first, but the pattern is the point.

**Nothing here is committed.** This describes the shape an integration should
take if we build one.

## Short answer on Tesla climate

Yes. Tesla's **Fleet API** is official and documented, and it covers climate
directly:

- `auto_conditioning_start` / `auto_conditioning_stop`
- `set_temps` (driver and passenger setpoints)
- `set_preconditioning_max` (defrost)
- `remote_seat_heater_request`, `remote_steering_wheel_heater_request`

Plus the rest of the obvious surface — lock/unlock, charge state and limits,
vehicle state.

### What it actually takes

⚠️ Verify all of this against current Tesla developer docs before planning around
it; this area has changed repeatedly and some of it I'm recalling rather than
checking.

1. **A Tesla developer account** and a registered application (client ID/secret,
   standard OAuth against the owner's Tesla account).
2. **Partner registration**, which requires **hosting a public key at a
   well-known URL on a domain you control** —
   `/.well-known/appspecific/com.tesla.3p.public-key.pem`.
3. **Signed vehicle commands.** Newer vehicles require Tesla's Vehicle Command
   Protocol rather than plain REST. Tesla publishes a Go SDK (`vehicle-command`)
   including an HTTP proxy that accepts ordinary REST calls and signs them —
   which is almost certainly what we'd run rather than implementing signing.
4. **Pricing.** Tesla introduced billing for Fleet API with a monthly free
   allotment. One person preconditioning their own car is very likely inside it,
   but confirm — an integration that silently costs money per tap is a bad
   surprise.

Note that (2) wants a public HTTPS endpoint, which sits awkwardly against the
"don't expose the sandbox" guidance in [claude.md](features/claude.md#auth). It's
only a static file, so host it somewhere else entirely — a static host or GitHub
Pages — and keep `sanctumd` unreachable as designed.

### The BLE alternative

The same vehicle command protocol also runs over **Bluetooth**, which is how the
phone key works. Sanctum could in principle send climate commands directly to the
car with no internet at all — genuinely better in a parking garage, and no API
billing.

Rejected for now: it means implementing a security-critical cryptographic
protocol in Swift when the reference implementation is Go, and enrolling a key
requires physical key-card access. That's a lot of exposure for a convenience
feature. Revisit only if the Fleet API path proves unusable.

## The pattern: integrations are agent tools, not tabs

Here's the part worth arguing about.

The obvious move is a Car tab. I think that's wrong, for three reasons:

1. **The tab bar is four items** and stays that way. We already merged Claude and
   Messages into Talk to hold that line
   ([architecture](architecture.md#navigation-model)).
2. **It fails the [vision](vision.md) test.** Preconditioning before you leave is
   genuinely useful. A screen showing your car's state of charge, which you open
   to look at, is a habit the phone manufactured. The first is a verb; the second
   is a dashboard, and dashboards are how this app becomes the thing it was built
   to replace.
3. **We already have a place for verbs.** Sanctum has an agent on a sandbox.

So: **`sanctumd` holds the credentials and exposes the car to Claude as a tool.**
"Warm up the car" is a sentence, not a screen. No new UI, no new tab, no state
you can idly check.

This gets better with scheduling. "Precondition at 7:40 on weekdays, but only if
there's something on my calendar" is a rule that lives on the sandbox and needs
no phone involvement at all — which is
[candidate feature #4](roadmap.md#candidate-features-post-v1), and this is the
first concrete case for it.

**The general rule, for everything that follows:**

> An integration earns a UI surface only when it needs to be *glanced at* under
> time pressure. Otherwise it's a capability, and capabilities belong to the
> agent.

Calendar earns a surface. Alarms earn a surface. A car does not.

### Where this could go wrong

Two honest risks.

- **Latency and trust.** "Warm up the car" through an agent is several seconds
  and a round trip, versus a button that is instant and visibly succeeded. If the
  agent path feels unreliable, people will want the button, and they'll be right.
  Mitigation: tool results render as a compact confirmation in the thread, not as
  prose.
- **Failure opacity.** A button that fails shows an error. An agent that fails
  may narrate something plausible instead. Tool calls must surface real status,
  and a failed command must never be reported as done — see the "no notification
  is ever a lie" rule in [notifications](notifications.md#rules).

## Candidate integrations, same treatment

All agent tools unless they argue their way to a surface: home automation
(HomeKit is local and would be the exception — it may genuinely warrant a few
buttons), weather, package tracking, transit times, anything with a REST API and
an OAuth flow.

The bar for each is the same question: **is this a verb or a dashboard?**

## Open questions

- **Does the Fleet API free tier cover personal use?** Determines whether this is
  free or a subscription with extra steps.
- **Which vehicles require signed commands**, and does the Go proxy cover all of
  them? Affects whether `sanctumd` gains a Go sidecar.
- **Does tool latency kill it?** Worth prototyping the agent-tool path before
  assuming it beats a button, because if it feels bad the whole pattern above is
  wrong and integrations do need surfaces.
