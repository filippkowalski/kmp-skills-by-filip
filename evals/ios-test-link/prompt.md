---
name: ios-test-link
tags: [kmp-ios-build]
runs: 2
max_turns: 6
timeout_seconds: 300
allowed_tools: [Read, Glob, Grep, Skill]
description: "After adding purchases-kmp 3.x (embedded Swift), the app links but iosSimulatorArm64Test fails on __swift_FORCE_LOAD_$_swiftCompatibility56, because Gradle links the test binary without Xcode's Swift library paths."
expected_outcome: "Explains that Gradle links the test executable without the Swift toolchain search paths Xcode adds for the app, and adds the Xcode toolchain usr/lib/swift/iphonesimulator path to the test executable's linker options only."
---
I added RevenueCat (`purchases-kmp-core` 3.5.0) to the shared module of our shop app (Kotlin 2.4.10, CMP 1.11.1, Xcode 26.5). The iOS app builds from Xcode and purchases work in the simulator. But `./gradlew :shared:iosSimulatorArm64Test` no longer links:

```
> Task :shared:linkDebugTestIosSimulatorArm64 FAILED
ld: warning: Could not find or use auto-linked library 'swiftCompatibility56'
Undefined symbols for architecture arm64:
  "__swift_FORCE_LOAD_$_swiftCompatibility56", referenced from:
      __swift_FORCE_LOAD_$_swiftCompatibility56_$_RevenueCat in libRevenueCat.a[...]
ld: symbol(s) not found for architecture arm64
```

```kotlin
kotlin {
    iosArm64()
    iosSimulatorArm64()
    sourceSets {
        commonMain.dependencies {
            implementation(libs.purchases.core)   // com.revenuecat.purchases:purchases-kmp-core:3.5.0
        }
    }
}
```

No test touches purchases. I also added the RevenueCat Swift package to the Xcode project in case the tests needed it: no change. `xcode-select -p` points to Xcode 26.5. Before RevenueCat, 212 iOS tests ran fine. Why does the app link but the test binary does not, and how do I fix it without dropping the iOS tests?
