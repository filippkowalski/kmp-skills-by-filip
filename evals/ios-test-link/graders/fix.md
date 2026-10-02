---
type: llm
weight: 1
focus: last_message
---
Context: the Gradle-linked iOS test executable fails to link after adding purchases-kmp 3.5.0 (embedded Swift, `libRevenueCat.a`), because it lacks the Swift library search path that Xcode adds for the app.

PASS if the fix adds the Xcode toolchain's Swift simulator library directory to the link of the test executable, for example:
`kotlin { iosSimulatorArm64 { binaries.withType<TestExecutable>().configureEach { linkTaskProvider.configure { toolOptions.freeCompilerArgs.addAll("-linker-option", "-L<xcode>/Toolchains/XcodeDefault.xctoolchain/usr/lib/swift/iphonesimulator") } } } }`
or `linkerOpts("-L.../Toolchains/XcodeDefault.xctoolchain/usr/lib/swift/iphonesimulator")` on the test binary. Taking `<xcode>` from `xcode-select -p`, resolved lazily, is best; a hardcoded Xcode path is acceptable. Scoping it to the test executable is expected. Advice to remove the extra RevenueCat Swift package from Xcode (the klib already embeds the SDK) is a fine extra.

FAIL if any of these is true:
- It disables or skips the iOS tests, or moves them to JVM-only.
- It removes purchases from the test link instead of fixing the link.
- It adds RevenueCat through SPM or CocoaPods as the fix.
- It changes the deployment target or framework type as the fix.
- It passes a flag that ignores undefined symbols (`-undefined dynamic_lookup` or similar), which hides the error and fails at runtime.
