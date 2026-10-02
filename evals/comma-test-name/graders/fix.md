---
type: llm
weight: 1
focus: last_message
---
Context: a backticked `commonTest` name with a comma breaks the Kotlin/Native test compile while the JVM test run stays green. The developer also asks how to stop this from slipping through again.

PASS if the answer does both:
- Renames the test so the name has no comma (for example "two offline edits of one note and the newest wins", or a camelCase name). Other test names with only spaces may stay as they are.
- Gives at least one concrete guard: run the Native test compile (`compileTestKotlinIosSimulatorArm64` or `iosSimulatorArm64Test`) before merge, or add a check (grep, lint or detekt rule) that rejects commas in backticked test names, or make the CI log show the compiler error.

FAIL if any of these is true:
- It ignores the test on iOS, excludes the file from the iOS test source set, or moves it to a JVM-only source set.
- It keeps the comma.
- It gives only a vague "run your iOS tests more often" with no concrete command or check.
