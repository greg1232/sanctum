# Email

The hardest of the four. iOS gives third-party apps **no access to the system
Mail account**. `MFMailComposeViewController` can present a compose sheet, but it
cannot read anything, and it depends on Mail.app being configured. To have mail
inside Sanctum we have to be a mail client.

## Options

| Approach | Verdict |
| --- | --- |
| `MFMailComposeViewController` only | Compose-only. No inbox. Fails the premise. |
| Provider REST APIs (Gmail API, Microsoft Graph) | Best UX per provider; OAuth is clean; but it's one integration per provider and each carries API-verification overhead. |
| **IMAP + SMTP** | Universal, works with app passwords, no vendor review. Painful protocol, but one implementation covers everyone. |
| JMAP | Excellent protocol; only Fastmail-class hosts support it. Nice-to-have, not a base. |

**Decision for v1: IMAP + SMTP**, with Gmail and Microsoft accounts connecting via
OAuth 2.0 over IMAP (`XOAUTH2`) rather than through their REST APIs. One code
path, real inbox, no provider lock-in. Revisit the Gmail API only if IMAP
throttling proves unworkable.

Open: hand-roll IMAP or take a dependency. This is the most likely exception to
the no-dependencies rule in [architecture](../architecture.md#platform-target).

## Sync model

- **Recent window only.** Default 30 days / 500 messages per folder, configurable.
  Sanctum is not a mail archive; search beyond the window hits the server.
- Folders in v1: Inbox, Sent, Archive, Trash. Nothing else is shown.
- **IDLE** for push while foregrounded (which, on a Guided Access phone, is most
  of the time). Background refresh via BGAppRefreshTask as a fallback.
- Outbound mail queues locally and sends on reconnect, visibly. A message that
  looks sent but isn't is the single worst bug this feature can have.

## Surfaces

1. **Inbox.** Threaded, sender-first, unread count bounded and honest. No badge
   on the tab above 99 — past that the number is noise.
2. **Thread.** Full message bodies. HTML rendered in a sanitized `WKWebView` with
   **remote content blocked by default** (tracking pixels are the norm, not the
   exception); one tap to load images for a given sender.
3. **Compose / reply.** Plain text first. Rich text only if it turns out to matter.
4. **Triage gestures.** Archive, trash, mark unread. Deliberately few.

## Security

- Credentials in Keychain, `WhenUnlockedThisDeviceOnly`, never iCloud-synced.
- TLS required; no downgrade, no "accept invalid certificate" affordance.
- Attachments open in `QLPreviewController`, never auto-open, never execute.
- Message HTML is sanitized before it reaches the web view, with JS disabled and
  a restrictive CSP. Assume every message is hostile input, because some are.

## Known gaps to decide later

Calendar invites arriving by mail (ICS) — parse into Calendar or ignore? Ignoring
is honest for v1; parsing is the kind of thing that makes the app feel finished.
