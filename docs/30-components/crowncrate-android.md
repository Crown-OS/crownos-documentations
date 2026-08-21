# crowncrate-android

**Status: Skeleton** · Kotlin · default branch `main` ·
[repo](https://github.com/Crown-OS/crowncrate-android)

The Android companion app for the CrownOS phone bridge.

> **It is an unmodified Android Studio "Compose + Navigation Suite" template with
> the app renamed.** One commit, `Created Android Project`. It has no networking
> code and no network permission, so it cannot talk to the desktop at all.

---

## Current contents

```
settings.gradle.kts     rootProject.name = "CrownCrate"
app/
  build.gradle.kts      namespace / applicationId = com.crownos.crate
  src/main/
    AndroidManifest.xml
    java/com/crownos/crate/
      MainActivity.kt
      ui/theme/{Color,Theme,Type}.kt
  src/test/…            ExampleUnitTest.kt        (template stub)
  src/androidTest/…     ExampleInstrumentedTest.kt (template stub)
```

`MainActivity.kt` is a `NavigationSuiteScaffold` with three placeholder
destinations — `HOME`, `FAVORITES`, `PROFILE` — and a body of
`Greeting(name = "Android")` rendering `Text("Hello Android!")`.

**`AndroidManifest.xml` has zero `<uses-permission>` entries.** No `INTERNET`, no
clipboard access, no notification listener, no call permissions.

---

## Toolchain

| | |
|---|---|
| AGP | 8.13.1 |
| Kotlin | 2.0.21 |
| compileSdk / targetSdk | 36 |
| minSdk | 29 |
| JVM target | 11 |
| Compose BOM | 2024.09.00 |
| UI | material3 + `material3-adaptive-navigation-suite` |

Requires Android SDK 36 and JDK 11 or newer. The Gradle wrapper is committed.

---

## Build

```bash
cd crowncrate-android
./gradlew assembleDebug
./gradlew installDebug     # to a connected device or emulator
./gradlew test
```

> **There is no ktlint plugin configured**, so `./gradlew ktlintCheck` and
> `ktlintFormat` do not exist and will fail with "task not found". Follow the
> official [Kotlin coding conventions](https://kotlinlang.org/docs/coding-conventions.html)
> by hand.

---

## What it needs to become

The desktop side defines the contract. To make the two halves talk, this app
needs:

1. **Permissions** — `INTERNET` at minimum; then
   `BIND_NOTIFICATION_LISTENER_SERVICE` for notification sync,
   `READ_PHONE_STATE` / `ANSWER_PHONE_CALLS` for the call bridge, and clipboard
   access.
2. **A TCP client** to `<desktop>:5252`, holding the connection open.
3. **CBOR encoding** of the message shape the daemon expects:
   ```
   Message { client: Ipv4Addr, method: Actions, body: Map<String, String> }
   Actions: CLIPBOARD | MEDIA | OPEN | OTPSYNC | MONITOR | VOLUME | SHUTDOWN
   ```
4. **Discovery** — the desktop binds UDP `:5253` for this, though its discovery
   function is currently a stub, so the protocol is undefined on both sides.
5. **Pairing and encryption.** Neither side has any. The desktop currently
   accepts unauthenticated commands including remote shutdown, which is not
   acceptable to ship.

See [IPC and protocols](../20-architecture/ipc-and-protocols.md#4-crowncrate--the-phone-bridge)
for the full protocol description, and
[crowncrate-linux](crowncrate-linux.md) for the server.

---

## Repository hygiene

The `.gitignore` is the stock Android Studio list, but `.idea/` is only partially
ignored — **nine `.idea/*.xml` files are committed**, along with
`gradle/wrapper/gradle-wrapper.jar`. The wrapper jar is conventional; the IDE
files are not.

---

## License

**No LICENSE file.** See
[Project status](../00-overview/project-status.md#licensing).
