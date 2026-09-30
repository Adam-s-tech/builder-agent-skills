---
name: ios-native
description: >
  Set up, verify, build, and run the iOS app on a simulator or device. Use when
  the user invokes /ios-native install, asks to install or configure Xcode,
  or asks to build, compile, run, launch, or deploy the iOS app; also use when
  they mention xcodebuild, xcrun, an IPA, a simulator, or native-run.
---

# iOS Native

If the user uses `/ios-native install`, follow [references/install.md](references/install.md).

## Build

Use `xcodebuild` to build the app. First, identify the scheme and workspace/project:

```bash
# List available schemes
xcodebuild -list
```

Build for simulator:

```bash
xcodebuild -scheme <SchemeName> -sdk iphonesimulator -configuration Debug -derivedDataPath build build
```

## Running on Simulator (preferred)

List available simulators:

```bash
xcrun simctl list devices available
```

If there is more than one option, ask the user which one to use. Open DeviceHub first, falling back to Simulator.app if it cannot open:

```bash
open -a DeviceHub || open -a Simulator
```

Build the app, boot the selected simulator if needed, then install and launch it with `simctl`:

```bash
xcodebuild -scheme <SchemeName> -sdk iphonesimulator -configuration Debug -derivedDataPath build build
xcrun simctl boot <SimulatorUDID>
xcrun simctl bootstatus <SimulatorUDID> -b
xcrun simctl install <SimulatorUDID> build/Build/Products/Debug-iphonesimulator/<AppName>.app
xcrun simctl launch <SimulatorUDID> <BundleIdentifier>
```

Skip `simctl boot` when the selected simulator is already booted. Read `<BundleIdentifier>` from the built app's `Info.plist` rather than guessing it.

## Running on a Physical Device

Use `xcodebuild` with `-sdk iphoneos` and deploy via `xcrun`:

```bash
xcodebuild -scheme <SchemeName> -sdk iphoneos -configuration Debug -derivedDataPath build build
xcrun devicectl device install app --device <DeviceUDID> build/Build/Products/Debug-iphoneos/<AppName>.app
```

## Finding the App Bundle Path

After a successful build, the `.app` bundle is located at:

```
build/Build/Products/Debug-iphonesimulator/<AppName>.app   # simulator
build/Build/Products/Debug-iphoneos/<AppName>.app           # device
```

## Gotchas

- Always run `xcodebuild -list` first to get the correct scheme name — do not guess it.
- If the project uses a `.xcworkspace` (e.g. CocoaPods), pass `-workspace <Name>.xcworkspace` instead of `-project`.
- Use `xcrun simctl list devices available` to get the exact simulator UDID; `native-run` may fail when Simulator.app is absent even if DeviceHub opens.
- Code signing is not required for simulator builds; for device builds, a valid provisioning profile and team ID are needed. If signing fails, inform the user.

## Shell command formatting

Always write commands on a single line — no backslash line continuations. The command ACL uses glob patterns without dotAll, so embedded newlines break matching for commands like `xcodebuild *`. For long output, pipe to `tail`/`head` or write to a file.
