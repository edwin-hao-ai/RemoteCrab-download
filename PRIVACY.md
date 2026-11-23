# RemoteCrab — Privacy Policy

_Last updated: 2026-10-23_

RemoteCrab is a local-network utility that turns your iPhone or iPad into a
camera, microphone, trackpad and keyboard for your Mac. This page
describes what data the app handles and what it does **not** do.

## Summary

RemoteCrab is **local-first**. On your local WiFi every feature is free and
complete, and the live media/input stream flows directly between your iPhone
and your computer over Bonjour + TCP. We do not collect analytics and we do
not require an account.

The optional **Remote Access** subscription is the one feature that uses our
servers: a small rendezvous service tells the two devices how to reach each
other, and a relay forwards bytes when a direct path is impossible. The relay
only forwards **application-layer encrypted (AEAD)** payloads; it never sees
your screen, microphone audio, keystrokes or clipboard. The LAN path does not
touch those servers at all.

## What data flows between the iOS app and the Mac app

When you tap the streaming button in the iOS app, the following data
is sent to the Mac over your local WiFi:

| Stream | Format | Purpose |
|---|---|---|
| Camera | H.264 video (1080p, 30 fps) | Live preview on the Mac; virtual webcam when wired up |
| Microphone | PCM audio (48 kHz) | Streamed to Mac speakers (and virtual mic when wired up) |
| Touchpad | Touch coordinates + modifier flags | Drives the Mac cursor via `CGEventPost` |
| Keyboard | Key codes + UTF-8 text | Drives Mac input via `CGEventPost` |

On the LAN the session is protected by your WiFi's WPA2/WPA3 encryption.
Once a session is paired, RemoteCrab also seals the stream with an
application-layer AEAD layer, so a relay (when you use Remote Access) only
ever forwards ciphertext.

## What RemoteCrab does **not** collect

- **No analytics** — no Firebase, no Mixpanel, no telemetry
- **No crash reporting** — the app does not phone home
- **No account** — no sign-up, no email, no user identifier
- **No recording** — the app streams in real time only; nothing is
  saved to disk or cloud unless you do so explicitly through the
  receiving app (e.g. Zoom's "Record to cloud" toggle)

## Remote Access servers and the subscription

Remote Access is optional and sold as an auto-renewable subscription through
the App Store. The app never sees your payment details; Apple handles the
purchase and gives the app a signed transaction, which our service verifies
before it hands out a remote address or relay token.

When Remote Access is used, the rendezvous service handles only connection
metadata:

| Data | Why |
|---|---|
| A stable computer id, computer name and platform | Tell your phone which computer answered |
| Candidate addresses (LAN/IPv6/NAT-PMP) | How to reach the computer |
| A relay token and a short-lived entitlement token | Authorise the relay/remote path |
| The App Store transaction's signed claims (bundle id, product id, expiry, revocation) | Verify the subscription on the server, not only in the app |
| An App Attest public key and Apple receipt (no biometric data) | Bind the subscription to this device |
| Source IP addresses in short-lived service logs | Operate/abuse protection |

The relay forwards only the same sealed media/input bytes described above. It
does not decrypt them, does not store them, and does not log their contents.
The rendezvous state (computer reachability, device public keys, subscription
binding) is kept on our server; the LAN-only experience never contacts it.

You can cancel the subscription in App Store settings. Cancelling does not
affect the free LAN features, and the service rejects expired/revoked
transactions.

## Permissions requested by the iOS app

| Permission | Why we need it |
|---|---|
| **Camera** | Live iPhone feed to your Mac |
| **Microphone** | Live iPhone mic to your Mac |
| **Local Network** | Bonjour discovery of your Mac on the same WiFi |

You can revoke any of these at any time in **iOS Settings →
Privacy**. RemoteCrab will continue to work for the streams whose
permissions are still granted.

## Permissions requested by the Mac app

| Permission | Why we need it |
|---|---|
| **Accessibility** | Drive the Mac cursor + keyboard from your iPhone (`CGEventPost`) |
| **Microphone** (optional) | Future: when the virtual-microphone extension is enabled |
| **Camera** (optional) | Future: when the virtual-camera extension is enabled |

macOS will prompt for Accessibility on first launch with a clear
explanation. You can re-prompt anytime via **System Settings →
Privacy & Security → Accessibility**.

## What the Windows app does on your PC

The Windows receiver is a separate program from the Mac app and is
covered by the same promises on the LAN: **nothing is uploaded, there is no
analytics and no telemetry.** If you use Remote Access, it registers with the
same rendezvous/relay service described above. It needs to touch the
following, all locally:

| What | Why | Where it goes |
|---|---|---|
| Reads and replaces your **clipboard** | Send text from the PC to the iPhone, and the other way | In memory; the OS clipboard is the only copy |
| Enumerates your **running applications** and their names/icons | The app switcher on the iPhone | In memory, sent to your iPhone only |
| Enumerates your **open windows** with a **JPEG thumbnail of each** | The window picker and app-window mirror on the iPhone | Thumbnails are generated on demand and sent to your iPhone only |
| Enumerates your **installed applications** | The "Open app…" launcher | In memory, sent to your iPhone only |
| **Injects keyboard and mouse input** into whatever window you are pointing at | That is the product | The OS input queue; nothing is recorded |
| Registers a **virtual camera** device (only if you turn it on with `--vcam`) | Appears as a camera to Zoom/Teams/OBS | A COM registration under `HKLM`, plus a frame ring in `%ProgramData%` |
| Reads its **own** settings and pairing token | Remember which iPhone is allowed, and reconnect to it | `%APPDATA%\RemoteCrab\tokens.json` |

It writes nothing outside `%APPDATA%\RemoteCrab\`,
`%ProgramData%\RemoteCrab\` and the registry entries above. The tray
menu can register it to start at login
(`HKCU\…\CurrentVersion\Run`); turn that off from the same menu.

**Not collected**: keystroke logs, screen history, window-title history.
The window list and the input events exist only for the life of the
process and are never written to disk. The "Why not connected" panel
prints network diagnostics (your local IP addresses, whether a VPN is
intercepting traffic) to your own screen and to no one else.

## Children

RemoteCrab is not directed to children under 13. We do not knowingly
collect any data from children. Because we do not collect any data at
all, this is largely moot.

## Changes to this policy

We may update this policy as the app evolves. Material changes will
be reflected by a new "Last updated" date at the top.

## Contact

If you have questions about this policy, contact us on GitHub at
[github.com/edwin-hao-ai/RemoteCrab](https://github.com/edwin-hao-ai/RemoteCrab).
