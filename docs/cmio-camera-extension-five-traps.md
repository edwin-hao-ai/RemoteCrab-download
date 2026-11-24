# Five things that fail silently when you ship a macOS camera extension

I spent three weeks getting a CMIO camera extension to the point where Zoom, Meet and OBS
list it and show live pixels from a phone. The extension worked for maybe four of those
days. The rest was spent on five failures that all share one property: **nothing throws,
nothing logs, and the thing looks like it is installed.**

If you are writing one now, this is the list I wish someone had handed me.

## 0. The shape that actually works

The host app pushes frames into the extension over a CMIO **sink stream**. The extension
forwards each sample buffer out of its **source stream**, which is what apps see.

```
host app  --sink stream-->  extension  --source stream-->  Zoom / OBS / Meet
```

There is no XPC between the app and the extension. `systemextensionsd` vends exactly one
mach service name (`CMIOExtensionMachServiceName`), so a custom `NSXPCListener` in the
extension cannot be looked up by the app at all. You get `No such process`, forever.
Apple's own answer on the developer forums is that app-to-CMIO-extension XPC is not
supported. Do not spend a day on it like I did.

## 1. `startService` returns, and then your process exits

```swift
CMIOExtensionProvider.startService(provider: provider)
// ...and then main() returns
```

`startService` registers the provider and returns immediately. If `main` then reaches the
end, the process exits, and the system is left with a registration pointing at nothing.
The device simply never appears, and there is no error anywhere.

The fix is one line that looks like a bug if you do not know why it is there:

```swift
CMIOExtensionProvider.startService(provider: provider)
CFRunLoopRun()
```

## 2. No transport type, no device

A device object with a name and two streams is not enough to be published. You have to
advertise `.deviceTransportType` in `deviceProperties` and return
`kIOAudioDeviceTransportTypeVirtual`. Without it the DAL has nothing to categorise you as
and the device does not show up, again with no diagnostic.

While you are there: `legacyDeviceID` should be `nil`. Leaving it set to anything makes
some version of the DAL do unhelpful things.

## 3. The DAL reports your sink stream as direction 0

This is the one that cost me the most time, because the header says the opposite of what
the runtime does.

`CMIOExtensionStream.StreamDirection`: `.source` means the extension produces frames and
`.sink` means it consumes them. The DAL reports an extension's **sink** stream as
**direction 0**, and its source stream as direction 1. OBS feeds frames into `streamIds[1]`.
So if you scan the client-side stream list for `direction == .sink` (which is 1 in the
enum), you will never find the stream that accepts your frames.

Hardcode what the runtime does, put a comment next to it explaining that the header and
the DAL disagree, and re-test it after every major macOS release.

## 4. Renaming or moving the app kills the activation record

The system stores the host app's origin path when the extension is activated. Move or
rename the app afterwards and you get an orphaned record that reports

```
[activated enabled]
```

and never launches. Everything looks correct in `systemextensionsctl list`. The extension
does nothing.

Deactivation requests fail with an authorisation error once ownership is broken, so you
cannot clean it up either. The only reliable way back is the replacement path: bump the
extension's `CFBundleVersion` and let the system replace it.

Which leads to the trap inside the trap.

## 5. Replacing the extension silently resets the user's approval

Every time you bump that version and re-register, the user's approval is dropped. If your
app auto re-registers whenever its binary fingerprint changes, you will wipe the approval
on every deploy, and the camera will vanish for the user while your logs say the
extension is enabled.

What I do now: only auto-repair when the **host app moved**. For a new build, the app
offers an explicit "Re-register" button and says that the user will have to re-approve it
in System Settings.

Two more things about that trip to System Settings: on macOS 15 and later the toggle lives
in **General > Login Items & Extensions > Camera Extensions**, not in Privacy & Security.
And deeplinking there is best effort only. Put the exact path in your UI text.

## How to prove it actually works

A command line process has no camera TCC. `AVCaptureDevice` requests from a CLI tool
return `notDetermined` and then zero frames, which looks exactly like a broken extension,
including for the built-in FaceTime camera. I lost an afternoon to proving my own bug that
did not exist.

Build a tiny ad-hoc signed `.app` with `NSCameraUsageDescription`, open it, grant it once,
and read pixel values. A number beats an opinion: a live phone feed through the extension
measured mean luma 111, and a black frame measures 16. Save a PNG as well, because a
screenshot settles arguments that a log line cannot.

## What is left over

Three weeks for a camera that appears in the list and delivers pixels. There is a second
extension for the microphone, which is a CoreAudio HAL plugin and a different set of
silent failures, but that is another post.

If you are stuck on any of the five, the symptom is almost always "installed, enabled, and
invisible", and the answer is almost always one of these.
