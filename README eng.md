[Uploading README Eng.md…]()
# Bluetooth Easy Talk · 蓝易聊

**A chat app where every message is a real `.txt` file sent over Bluetooth.**

No server. No account. No cloud. Just files, moving from one phone to another.

---

## Status at a Glance

| Platform | Version | Status |
|---|---|---|
| **Android** | `0.0.12` | ✅ Released · builds, signs, installs, tested |
| **HarmonyOS (native)** | `0.0.12` (source) | ⛔ **Blocked — source complete, does not compile yet** |

- Android artifact: `releases/v0.0.12/bluetooth-easy-talk-0.0.12.apk` — **74,492 bytes (72.7 KB)**
- Android requirements: `minSdk 14` → **Android 4.0 and newer**
- HarmonyOS: 14 ArkTS files, 2,103 lines written — **11 compiler errors remaining, awaiting resolution**

---

## The Core Idea

There is no database anywhere in this app. The chat history is literally the
contents of two ordinary folders:

```
<app folder>/txtchat/
├── outbox/    every message you sent, one .txt file each
├── inbox/     every message you received, one .txt file each
└── *.txt      exported transcripts
```

**The file name carries the metadata; the file body is the message.**

```
outbox/20240601-120430_MyPhone.txt    body: See you at the station at seven
inbox/20240601-120502_XiaoMing.txt    body: OK, I'll be ten minutes early
```

Which means:

- Any `.txt` opens in any text editor, **without this app**
- Delete a file, delete a message — nothing is hidden in a database
- The file that arrives is **byte-for-byte identical** to the file that was sent

---

## Feature by Feature

### 01 — Every message is a real `.txt` file

- Sending a message writes it to a plain text file first, then pushes *that file* over Bluetooth.
- The received copy is byte-for-byte identical: not rewritten, not compressed, not wrapped in a container.
- Any text editor can open it. This app is not required to read your own history.

### 02 — Your chat history *is* two folders

- `outbox/` holds what you sent, `inbox/` holds what you got. One file, one message.
- There is no "export your history" step, because the history is already in a folder you can open.
- Delete a `.txt` from a file manager and the message disappears from the chat. No hidden state.

### 03 — Direct Bluetooth, with no network capability at all

- The app **never requests the INTERNET permission**. This is not a promise about behaviour — at the OS level it simply cannot reach a network.
- No account, no sign-up, no server, no cloud sync, no telemetry.
- Data exists only on the Bluetooth link between two paired phones.
- It works in aeroplane mode.

### 04 — A wire protocol built for `.txt`

- Standard Bluetooth serial port (RFCOMM / SPP), service UUID `00001101-0000-1000-8000-00805F9B34FB`.
- One message = one frame: a header line carrying the byte count and file name, followed by the raw file bytes.
- Framing is **length-prefixed**, so newlines, blank lines, or a message body that happens to look like a frame header cannot corrupt the stream.
- 256 KB per message. Malformed headers and truncated frames are rejected loudly instead of failing silently.

### 05 — Flat design, zero icons

- Not a single icon in the interface. Every "button" is text, distinguished by accent colour and font weight.
- Message bubbles are flat rounded rectangles: no shadows, no gradients, no outlines, no ripples.
- **The APK contains zero image assets** — verified. The whole UI is type, colour blocks and 1 dp hairlines.
- Even the chat backdrop is not shipped with the app; it is a picture you choose at runtime.

### 06 — Extremely lightweight

- **72.7 KB** installed size — less than a tenth of an ordinary phone photo.
- **Zero third-party dependencies**: no AndroidX, no AppCompat, no Material, no image loader, no JSON library, no minifier.
- 16 Java files, ~3,072 lines. Short enough to read end to end.
- Background memory use is tiny, because it only does one thing: receive a file, store a file.

### 07 — Runs on Android 4.0 and newer

- `minSdk 14`. A 2011 handset and today's flagship both install it.
- New APIs are deliberately avoided: no `setBackground` (API 16), no `setShowWhen` (API 17). Nothing crashes on old releases with a `NoSuchMethodError`.
- Notifications fall back to the pre-API-16 construction path on Android 4.0 / 4.1.
- Storage lives in the app-specific folder only, so on Android 10+ it needs **no storage permission at all**.
- Signed with v1 + v2 + v3; the v1 JAR signature is what makes Android 4.x accept it.

### 08 — Dark, light, or follow the system

- One tap in Settings. It applies immediately, without restarting the app.
- Two complete palettes with eight colour slots: background, surface, body text, muted text, accent, sent bubble, received bubble, hairline.
- The dark mode is hand-rolled rather than delegated to AppCompat — which is why it also works on **Android 4.0**, not only on recent releases.

### 09 — Four languages

- **中文 · English · 日本語 · Русский**, switchable inside the app, or set to follow the system locale.
- **92 strings per language**, with all four key sets verified identical — no string silently falls back to English.
- Language names are written in their own script (中文 / English / 日本語 / Русский), so you can find yours even when the current UI is unreadable.
- The app name itself is localised: **蓝易聊** on Chinese systems, **Bluetooth Easy Talk** everywhere else.

### 10 — A custom chat background

- Pick your own picture to sit behind the message list; remove it whenever you like.
- A 60 % wash of the current theme background goes over it, so dark type on the light theme and light type on the dark theme both stay readable.
- With a backdrop set, message bubbles turn slightly translucent so the picture shows through.
- Guarded for old hardware: two-pass decode with a screen-sized sample, `RGB_565` at two bytes per pixel, and `OutOfMemoryError` caught. It survives a phone with a 16 MB heap.

### 11 — Quiet in the foreground, reliable in the background

- A message arriving **while you are looking at the chat** raises nothing at all. It appears in the list and the view scrolls down — a popup would only be noise.
- A message arriving **while you are in another app** raises an ordinary system notification: sender as the title, message as the body, tap to return, swipe to dismiss.
- The two kinds live on **separate notification channels**, so either can be silenced independently from system settings.
- With "background receive" enabled, a foreground service holds the Bluetooth link open, so leaving the app does not drop the connection.

### 12 — Works even if the other side has no such app

- Real-time chat needs the app on both ends. Two other paths do not:
- **"Share .txt over Bluetooth"** hands a transcript or a single message to the system Bluetooth stack (OBEX Object Push). The file lands in the peer's Bluetooth received folder and any app on their phone can read it.
- A `.txt` someone sends you through system Bluetooth can be **shared into this app** from a file manager or the notification, and it is imported as a normal chat message.
- So it works both as a chat app and as a way to pass notes over Bluetooth.

### 13 — Builds without Gradle

- The build script drives `aapt2 → javac → d8 → zipalign → apksigner` directly. No dependency download, no Gradle daemon, no Android Studio.
- The toolchain (JDK 17 + Android build-tools 34 + platform 34) is fetched once by a script into `.tools/` inside the repository.
- Each build writes to its own `releases/v<version>/` folder, so a previously built APK is never overwritten.

### 14 — The wire protocol is unit-tested

- The codec is deliberately isolated in a class with no `android.*` import, so **the very same source file** that ships in the APK also runs under a desktop JVM. The tests cover the real code, not a copy of it.
- **70 assertions, all passing**, covering header parsing, rejection of malformed frames, zero-length files, over-long headers, truncated frames, and file names containing spaces and underscores.
- Sender names in all four languages survive a round trip, and file names are verified to stay within filesystem limits.
- Security cases: path traversal (`../../secret.txt`), illegal Windows characters, control characters and dots-only names are all neutralised.
- Plus a full loopback test: frames are written the way the sender does and read back the way the receiver does, then compared byte for byte — including CJK and Cyrillic text, emoji, multi-line bodies, and a message whose body deliberately imitates a frame header.

### 15 — Almost no permission burden

- No INTERNET, no location, no contacts, no camera, and no storage access on modern Android.
- On Android 12+ it asks for the two Bluetooth runtime permissions. Older releases use the install-time Bluetooth permissions, with the applicable maximum version pinned in the manifest.
- The notification permission is requested only at the moment you switch on "background receive", instead of nagging every time you launch.
- External-storage write access is capped at Android 4.3 and below, so recent devices never see it at all.

### 16 — Export the whole conversation as one `.txt`

- One tap in Settings merges `inbox` and `outbox` into a single timestamped transcript.
- The result is plain text: share it, print it, archive it, or send it on over Bluetooth again.
- Exports are shared through a small read-only content provider built into the app, so Android 7+ never trips its file-URI exposure guard.

### 17 — The keyboard is already open when you need it

- Opening the app focuses the message field and raises the system IME straight away — no extra tap before the first word.
- Implemented by declaring `SOFT_INPUT_STATE_VISIBLE` on the window and re-requesting focus one frame later, because `showSoftInput` is silently ignored when the window has not yet taken focus.
- The window uses `ADJUST_RESIZE`, so the keyboard pushes the message list up instead of covering it.

### 18 — Sending does not interrupt typing

- After a message is sent the field is cleared but **keeps focus**, and the keyboard stays open — you can keep typing without tapping the field again.
- The SEND control is deliberately not focusable, so tapping it can never steal focus from the input.
- A defensive re-focus runs after every send as well, because some vendor ROMs behave differently from stock Android here.

---

## Three Ways to Use It

**1. Real-time chat** — the app is needed on both ends.
Pair the two phones once in the system Bluetooth settings. On one phone: **DEVICES → START LISTENING**. On the other: **DEVICES → tap the peer's name**. Once connected, just type and send.

**2. Send a `.txt` to anyone** — no app needed on the other side.
Settings → **SHARE OUTBOX .TXT** → pick Bluetooth in the system share sheet → choose the peer device. The file lands in their Bluetooth received folder.

**3. Accept a `.txt` someone sent you** — no app needed on their side.
They push the `.txt` to you using system Bluetooth. From the notification or a file manager, choose **Share to Bluetooth Easy Talk**. It is imported as an ordinary chat message.

---

## Technical Specifications

| Item | Value |
|---|---|
| Package name | `io.txtbt.chat` |
| Android version | `0.0.12` (versionCode `12`) |
| APK size | 74,492 bytes (72.7 KB) |
| Min SDK | 14 (Android 4.0) |
| Target SDK | 33 (Android 13) |
| Compile SDK | 34 |
| Transport | Bluetooth RFCOMM (SPP) |
| Service UUID | `00001101-0000-1000-8000-00805F9B34FB` |
| Message limit | 256 KB |
| Encoding | UTF-8 |
| Signing | v1 + v2 + v3 |
| dex version | 035 |
| Third-party libraries | none |

**Source size:** 16 Java files / ~3,072 lines · 5 layouts · 92 strings × 4 locales · **0 image assets**

**APK contents:** `classes.dex` 71,728 B · `resources.arsc` 27,452 B · binary manifest · 5 binary layouts · signature files

---

## HarmonyOS Native Version — Development Status

**Short version: the source is written, but it does not compile yet. Work is paused pending a resolution.**

This is reported plainly rather than omitted, because a README that promises a
build you cannot produce is worse than one that tells you where it stands.

### What is done

- **The toolchain is proven.** DevEco Studio 26 (HarmonyOS SDK 26.0.0 / API 26), hvigor 6.26.8, Node and the HAP signing tools are all located and driven from the command line. An empty HarmonyOS project was built end to end, producing a valid `entry-default-unsigned.hap` (45,688 bytes) — so the pipeline itself works.
- **All 14 ArkTS source files are written — 2,103 lines** — covering the same architecture as the Android build: the length-prefixed `.txt` protocol (`TxtCodec.ets`), the two-folder store (`ChatStore.ets`), the Bluetooth SPP engine (`ChatEngine.ets`), preferences, theme, custom background, a permission helper, and three pages (chat / devices / settings).
- **Protocol compatibility is designed in.** Both sides use the same standard SPP UUID and the same frame layout, so a HarmonyOS build — once it compiles — should interoperate with the Android build over the same link.

### What is blocking it

The ArkTS compiler rejects the project. The error count has been driven down
**50 → 19 → 11**, but the remainder have not been resolved:

1. **11 remaining ArkTS errors**, all of the same class: *"Unable to resolve signature of property decorator when called as an expression."* They are almost certainly one root cause cascading across the three page files, rather than eleven independent bugs.
2. **The build log truncates the diagnostic.** The `At File: …` suffix is cut off at a fixed width, so the errors currently come with no file or line number — which turns each attempt into an educated guess rather than a fix.
3. **Signing is a second, independent blocker.** Retail HarmonyOS phones will not install a HAP signed with the OpenHarmony materials bundled in the SDK; that requires a certificate and provisioning profile issued through a Huawei developer account. The locally built artefact can only be used with an emulator or an OpenHarmony device.

### What is not lost

The Android build is complete, tested and installable. The two codebases share
one protocol and one data model, so the HarmonyOS work is not wasted — it needs
a compiler-level fix and a signing path, not a redesign.

### Planned next steps, when work resumes

- Isolate the remaining ArkTS error class into a minimal reproduction page, then re-introduce features one at a time.
- Re-verify on the DevEco emulator once the project compiles.
- Decide on the distribution route for HarmonyOS phones (Huawei AGC signing vs. sideloading on OpenHarmony hardware).

---

## What Changed in 0.0.12

Two keyboard behaviours, both requested and now shipped:

- **The system IME opens by itself on launch** — the app no longer waits for a tap on the field.
- **The input stays ready after sending** — sending a message no longer collapses the keyboard or drops focus.

Nothing else was altered. The file format, the wire protocol and the folder layout
are unchanged, so `0.0.12` and `0.0.11` remain fully interoperable — and so does
the planned HarmonyOS build.

---

## Known Limitations

1. **Real-time chat requires the app on both ends.** Android does not let a third-party app watch the system Bluetooth received folder, so when the other side has no app you are limited to the share / import paths above; those messages do not appear in the conversation automatically.
2. **The phones must be paired first.** Pairing happens in the system Bluetooth settings; the app does not implement its own pairing flow.
3. **One peer at a time.** No group chat. After a disconnect, tap the peer's name again to reconnect.
4. **Message times come from the local file write time.** If the two phones' clocks disagree, ordering stays correct but the displayed timestamps will differ.
5. **Background notifications depend on the "background receive" switch.** With it off, the app disconnects as soon as it leaves the screen, so there is nothing to notify about.
6. **The custom backdrop affects the message area only.** The top bar, status line and input row stay flat theme colour.
7. **This is a text chat tool, not a file transfer tool** — 256 KB per message.
8. **The current APK is signed with a self-signed debug certificate** whose password is `android`. Fine for personal use; use your own key for any real distribution.

---

## Where Everything Lives

```
releases/v0.0.12/bluetooth-easy-talk-0.0.12.apk   current Android build
releases/v0.0.11/bluetooth-easy-talk-0.0.11.apk   previous build, kept intact
dist/txtchat-1.0.0.apk                            first build, kept intact
app/src/main/                                     Android sources
harmony/                                          HarmonyOS sources (does not compile yet)
build.ps1                                         Android build script
tools/fetch-tools.ps1                             one-time toolchain download
tools/run-tests.ps1                               protocol test suite
README Eng.md / README CN.md                      this documentation
```

**Install:**

```powershell
adb install -r releases\v0.0.12\bluetooth-easy-talk-0.0.12.apk
```

**Build from source:**

```powershell
pwsh -File tools/fetch-tools.ps1      # one-time toolchain download
pwsh -File build.ps1 -Clean           # produces dist output
```

---

## License

For learning and personal use. Modify freely.
