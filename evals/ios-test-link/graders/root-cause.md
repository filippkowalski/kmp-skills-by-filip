---
type: llm
weight: 2
focus: last_message
---
Context: after adding `purchases-kmp-core` 3.5.0, the iOS app builds and runs from Xcode, but `linkDebugTestIosSimulatorArm64` fails with an undefined `__swift_FORCE_LOAD_$_swiftCompatibility56` referenced from `libRevenueCat.a`. Adding the RevenueCat Swift package in Xcode did not help.

PASS if the answer states all of these:
- The iOS test executable is linked by the Kotlin/Native toolchain from Gradle, not by Xcode.
- When Xcode links the app, it adds the Swift toolchain library search paths (and SPM frameworks). The Gradle test link does not get them.
- The purchases klib embeds a compiled Swift library (`libRevenueCat.a`) that auto-links Swift compatibility libraries such as `swiftCompatibility56`. These live in the Xcode toolchain's `usr/lib/swift/iphonesimulator` directory, which the test link never searches.

FAIL if any of these is true:
- It blames a missing SPM or CocoaPods dependency.
- It blames the selected Xcode, stale caches or `~/.konan`.
- It blames the deployment target, or says the SDK does not support the simulator.
- It says the tests must mock purchases, without explaining the link difference.
- It blames static versus dynamic framework settings (the test executable is not the framework).
