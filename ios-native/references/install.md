# Install and verify iOS tooling

Treat `/ios-native install` as a setup and readiness check, not a request to build the app. Run these checks on the machine executing the commands, not on an assumed user Mac:

```bash
uname -s
command -v xcodebuild
xcode-select -p
xcodebuild -version
xcodebuild -showsdks
xcrun simctl list runtimes
xcrun simctl list devices available
command -v node
command -v npx
```

Check for the Xcode GUI and Simulator GUI independently with `test -d /Applications/Xcode.app` and `test -d /Applications/Xcode.app/Contents/Developer/Applications/Simulator.app`; Xcode command-line tools working does not mean Simulator.app is present. Also check `test -d /Applications/DeviceHub.app`, since DeviceHub is the preferred way to open a simulator window and Simulator.app is only the fallback. Locate the project's `.xcodeproj` or `.xcworkspace`, run `xcodebuild -list -project <Project>.xcodeproj` (or `-workspace`), then compare its deployment target with an available simulator runtime. Do not guess a scheme or target ID.

If any check fails, give the user only the relevant next steps and repeat the failed check after they complete them:

- macOS is required for Xcode and iOS Simulator. On non-macOS hosts, explain that installation cannot be done there and guide the user to a Mac.
- If full Xcode is missing, direct the user to install **Xcode** through the Mac App Store, open it once to complete first-launch components and license prompts, and select its Command Line Tools in Xcode Settings → Locations. `xcode-select --install` installs only Command Line Tools, not the full Xcode app or Simulator.
- If the active developer directory points to Command Line Tools or the wrong Xcode, explain the mismatch. Ask before running `sudo xcode-select -s /Applications/Xcode.app/Contents/Developer` or any other system-wide/elevated change; otherwise guide the user through selecting Xcode in its Settings → Locations.
- If the target iOS runtime is missing or older than the app's deployment target, guide the user to install a compatible iOS simulator runtime from Xcode Settings → Platforms (or Components, depending on Xcode version), then select/create a simulator in Xcode's Devices and Simulators window.
- If both `DeviceHub.app` and `Simulator.app` are missing even though `xcodebuild` works, do not claim that `open -a DeviceHub`, `open -a Simulator`, or `simctl` can show a window. Explain that the Xcode installation needs its Simulator component repaired/reinstalled through Xcode or the Mac App Store. `simctl` may still run an app headlessly, but cannot make its screen visible to the user here.
- If Node.js/`npx` is missing, guide the user to install Node.js using their preferred package manager or the official Node.js installer, then rerun the checks. Do not silently install packages or use an unapproved installer.

When prerequisites pass, verify the actual project with the Build and Running on Simulator steps in the main skill: build for a compatible simulator destination, find the resulting `.app`, then launch it. Ask which device to use if several are available. Open DeviceHub first, falling back to Simulator.app if it cannot open. If neither GUI is available, `simctl boot`, `simctl install`, and `simctl launch` can verify a headless run instead, but do not describe it as a visible launch. Report any command that could not be verified rather than claiming the setup is complete.

Finish with a concise pass/fail summary for Xcode, project scheme, compatible runtime, build, DeviceHub/Simulator GUI, and launch, stating whether GUI launch or only headless `simctl` is possible. A remote Builder browser preview does not display a native macOS Simulator or DeviceHub window; opening Xcode on a remote host does not make it visible to the user.
