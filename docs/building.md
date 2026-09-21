# Building & deploying

Sanctum is a personal app for one or two phones. It is not going to the App
Store, which removes review from the picture but makes **code signing** the
central problem rather than an afterthought.

## What you need

| Thing | Cost | Why |
| --- | --- | --- |
| **Apple Developer Program** | $99/yr | Non-negotiable. See below. |
| A Mac with Xcode | — | No way around it for iOS. |
| An iPhone running iOS 26+ | — | AlarmKit. A second-hand handset is fine. |
| A Linux box for `sanctumd` | $0–20/mo | Home server, VM, or a small cloud box. |
| Tailscale or WireGuard | free tier | Reaching the sandbox without exposing it. |
| A domain | ~$15/yr | Only if you do the [Tesla integration](integrations.md). |

The $99 is required, not optional, and it's worth listing what it actually buys
because three separate decisions depend on it: **push notifications** (APNs, so
the screen can sleep), **the AlarmKit entitlement** (so alarms work at all),
**Time Sensitive notifications** (so roster messages break through Focus), and
**one-year provisioning** instead of seven-day.

## Signing: the part that matters

| Route | Profile life | Verdict |
| --- | --- | --- |
| Free Apple ID ("Personal Team") | **7 days** | Fine for the first week of hacking. Unusable beyond that, and no push or AlarmKit. |
| **Paid — development or Ad Hoc** | **1 year** | **This.** Register the device UDID, install from Xcode or an IPA. |
| TestFlight | **90 days** | Tempting, but builds expire quarterly and a dead build on a locked phone is a bad morning. |
| Enterprise Program | 1 year | $299/yr, restricted to large organizations, for internal apps. Not us. |
| App Store | — | Review, for an app with an audience of one. No. |

### The expiry landmine

**A provisioning profile that expires while the phone is locked into Sanctum
means the app will not launch.** On a device whose entire premise is that you
can't leave the app, that's the single worst operational failure available.

Two mitigations, both cheap, and the first one should be built:

1. **The app checks its own expiry.** `embedded.mobileprovision` ships inside the
   bundle and can be parsed at runtime for its `ExpirationDate`. Sanctum should
   read it at launch and start warning at 30 days out — visibly, persistently,
   escalating. This is a small amount of code that prevents the worst class of
   outage this project has.
2. **A calendar event at 11 months**, set the day you first install. Sanctum has
   a calendar; use it.

Also surface build version, build date and profile expiry in settings. On a
device you can't easily inspect, the app should be able to tell you what it is.

## Repo and build layout

```
sanctum/
  Sanctum.xcodeproj          checked in
  Sanctum/                   app target: entry point, tab shell, assets
  Packages/
    SanctumKit/              design system, storage, keychain, clock, net
    Calendar/  Email/  Talk/  Alarm/
  Sandbox/
    sanctumd/                the backend
  docs/
```

Local SPM packages referenced from the project. Keep `.xcodeproj` checked in and
plain — project generators (Tuist, XcodeGen) solve merge conflicts that a solo
repo doesn't have yet. Revisit if more than one person ever touches it.

Per [architecture](architecture.md#layering-rules), feature packages never import
each other and `SanctumKit` imports no feature. That boundary is what keeps the
app buildable and previewable without a sandbox, mail host, or calendar
permission.

## Building

```bash
# tests, no signing needed — simulator only
xcodebuild test \
  -scheme Sanctum \
  -destination 'platform=iOS Simulator,name=iPhone 16'

# package tests, much faster, run these constantly
swift test --package-path Packages/SanctumKit
```

Device builds go through Xcode with the device registered to your team. Archive
and export Ad Hoc only if you want to install without a cable.

## CI

GitHub Actions on a macOS runner, running **tests only**. Do not build a release
pipeline: signing in CI means managing certificates in CI, and this app installs
onto one phone from one Mac. The pipeline would cost more than it saves.

What CI should catch:

- Unit tests across all packages.
- Snapshot tests in light/dark at the largest Dynamic Type size.
- The long-uptime soak test from [power](power.md), which can run in a simulator
  against a fake clock.

## The sandbox

### ⚠️ Open: what language is `sanctumd`?

The dependencies have quietly forced this and the docs haven't acknowledged it.
We've accumulated:

- **whatsmeow** — Go ([messaging](features/messaging.md))
- **Tesla `vehicle-command`** — Go ([integrations](integrations.md))
- **signal-cli** — Java, but driven as a JSON-RPC subprocess, so language-neutral
- **Claude Code / Agent SDK** — a CLI, plus SDKs in TypeScript and Python

Two coherent answers:

- **Go core.** Native for the two libraries that matter, excellent as a
  long-lived network daemon, single static binary to deploy. Claude Code runs as
  a subprocess.
- **TypeScript core.** Native for the Agent SDK, which is the most
  interesting surface. The Go libraries become sidecar processes.

I lean **Go**: `sanctumd` is fundamentally a long-running daemon holding
sockets, and driving a CLI as a subprocess is easier than reimplementing a
WhatsApp client. But M3 is the Claude milestone and it arrives first, so this
should be decided with a spike then, not assumed now.

### Deploying it

Whatever it's written in: a single service under systemd, bound to **loopback
only**, reachable over Tailscale. Never a public port.

```
/opt/sanctum/sanctumd          binary
/var/lib/sanctum/              SQLite, transcripts, message store, tokens
/etc/sanctum/config.toml       roster, accounts, APNs key path
```

- **Full-disk encryption on the host.** Non-negotiable — the bridge decrypts your
  messages, so this box holds plaintext history for both networks
  ([why](features/messaging.md#what-this-costs-you)).
- `.p8` APNs key and OAuth tokens in `/etc/sanctum`, mode 0600, never in git.
- Back up `/var/lib/sanctum`. It holds Claude transcripts and message history you
  can't re-fetch.

## First deployment to the phone

Order matters — several of these are impossible once the lockdown is on.

1. Register the device UDID with your developer team.
2. Build and install from Xcode.
3. Enrol the device with `sanctumd`
   ([Secure Enclave keypair + one-time code](features/claude.md#auth)).
4. **Run the full [configure before you lock](guided-access.md#configure-before-you-lock)
   checklist** — accounts, wifi networks, Wallet, phone key, Signal/WhatsApp
   linking, and the [power settings pass](power.md#what-only-pre-lockdown-configuration-can-do).
5. Work through the [verification matrix](notifications.md#verification-matrix)
   while you still have a normal phone to compare against.
6. Enter Guided Access.

## Updating a locked phone

With Guided Access (option A) this is easy and manual: triple-click, enter the
passcode, plug into the Mac, build and run, re-enter Guided Access. A minute,
whenever you want.

Worth knowing before choosing a lockdown mode: under **device-level Single App
Mode** ([option C](guided-access.md#c--single-app-mode-device-level-app-lock)),
updates have to go through MDM app distribution, which is a real pipeline. That's
another reason to develop against A and treat B/C as a destination.

## Annual maintenance

Once a year, in one sitting:

- Renew the Apple Developer Program.
- Regenerate certificates and provisioning profiles.
- Rebuild and reinstall the app; confirm the new expiry in settings.
- Re-link Signal and WhatsApp if the companion sessions have lapsed.
- Rotate the sandbox device key.

Put it on the calendar the day you first install. The app will also be warning
you by then, if the expiry check above got built.
