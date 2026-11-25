# A desktop in a browser tab: five things that broke, and what I stopped doing

I built a thing that lets you open a Mac or Windows PC in a browser tab on any other device:
no client to install, no account, no meeting link. You press "share this computer" on the
machine, get a six digit code and a URL, and the person on the other side pastes it.

It works. Getting there cost me five separate failures that all looked like the same bug
("the screen is black") and had nothing to do with each other. This is the list I wish I
had, plus the one number that finally settled an argument.

## 1. Two milliseconds of silence make Opus decode to mono

The first version sent audio as stereo Opus. On the receiving side the audio came out
mono, thin, and about a third of the volume. Not quiet exactly, just wrong in a way that
is hard to describe until you hear it.

The cause: Apple's `AudioConverter` Opus decode path is sensitive to how you feed it. When
the input proc runs out of data, the obvious thing is to return `noErr` with zero packets.
That call finalises the decoder **permanently**. Every later fill comes back empty, and
what you hear is the converter's idea of a mono downmix of whatever it did before.

The fix is to return a real error code when the current fill is drained, and let the
converter ask again:
`kIBOpusInputDrained`. One enum case, three days of confusion.

Measurements, since "it sounds thin" is not evidence: stereo samples came back at roughly
one third of the source amplitude, with both channels identical.

I stopped using Opus over the LAN entirely after that. PCM at 48 kHz stereo is 1.5 Mbps,
which next to H.264 video is noise, and it has **zero** codec latency. The bandwidth
argument for Opus only starts to matter on a relay, and that is a different trade.

## 2. A 2 second poll leaked a file descriptor, and 31 hours later the video stopped

This one is my favourite because the symptom was so far from the cause.

After about a day of uptime, screen sharing went black. The phone showed "waiting for your
computer". Clicks still worked. No errors anywhere in the app.

The machine had **4,830 leaked `AF_UNIX` sockets** and every new one failed with `errno 24`.
Under that pressure `VTCompressionSessionCreate` returns **-12908**, which the app read as
"encoder unavailable" and skipped the frame. The encoder was fine. The process had simply
run out of descriptors, because a device watcher called `listDevices()` every 2 seconds and
the successful path forgot to `cancel()` the connection.

Reproduced on the bench: 20 polls, descriptors 4 → 25. After adding `defer { conn.cancel() }`,
4 → 5. The regression test asserts that 10 polls add at most 2 descriptors.

I also stopped retrying the encoder per frame. Two failures in a row now means the session
declares itself unavailable, says why, and stops. A retry loop at 24 fps is not a fix, it
is a way to hide the error.

## 3. The sink stream's direction is the opposite of what the header says

If you ever write a CoreMediaIO camera extension: the DAL reports an extension's `.sink`
stream as direction **0**, and `.source` as **1**. The header's documentation says the
opposite of what the runtime does. OBS feeds `streamIds[1]`.

I hardcoded the runtime's answer with a comment explaining that the header disagrees, and
I re-test it after every major macOS release.

Related, and cheaper to learn from me than to discover: `CMIOExtensionProvider.startService`
returns immediately. If `main` then falls off the end, the process exits and the system is
left with a registration pointing at nothing. `CFRunLoopRun()` after it is not optional.

## 4. Renaming the app orphaned the extension, and it still reported "enabled"

The system stores the host app's origin path when a system extension is activated. Move or
rename the app and you get a record that prints

```
[activated enabled]
```

and never launches anything. Deactivation requests fail with an authorisation error, so you
cannot clean it up either. The only way back is the replacement path: bump the extension's
`CFBundleVersion` and let the system replace it.

Which resets the user's approval. Every time. If your app auto re-registers when its binary
fingerprint changes, you will silently drop that approval on every deploy, and the user's
camera will vanish while your logs say the extension is enabled.

What I do now: auto-repair only when the app **moved**. A new build gets an explicit
"Re-register" button that says the user will have to approve it again, and the exact path
in System Settings, because on macOS 15 and later it lives under General → Login Items &
Extensions → Camera Extensions, not Privacy & Security.

## 5. The six digit code is the security model, so it has to expire

The link is meant to be read out loud or pasted into a chat. That means the code must die
when the share ends, not when the process dies.

Two rules that came out of testing it:

- Closing the share invalidates the code **immediately**, including for a viewer that is
  mid-handshake. A code that survives the share is a code someone can brute force later.
- The viewer's token is bound to that one share. Reusing a cached relay token after a
  reconnect produced a silent "connection failed" that the app interpreted as the peer
  taking over. Now a dial without a fresh lookup refuses to use a cached token, and says
  "looking up" instead of pretending to connect.

Symmetric NAT means the direct path fails and everything goes through a relay. That is the
one part that costs real money, which is why remote access is the paid tier and the local
network is free. I would rather say that plainly than pretend a server is not involved.

## What I stopped doing

- Retrying things that fail at frame rate. Two failures, then a message with a reason.
- Trusting a number I did not measure. A "6220 kbps" in a stream header turned out to be
  computed, never measured: asking for 6,220 and 9,331 produced byte-identical output.
- Adding a setting for a failure. Every one of the five above was fixed by removing a
  cause, not by adding a switch.

The product is at vgoapp.com/remotecrab if you want to see the shape of it. The browser
half is free on a local network, and the viewer really is just a tab.
