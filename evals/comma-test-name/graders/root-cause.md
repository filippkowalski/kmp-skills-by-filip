---
type: llm
weight: 2
focus: last_message
---
Context: a new `commonTest` file compiles and runs on the JVM (`testDebugUnitTest` green) but `compileTestKotlinIosSimulatorArm64` fails, and the error text was cut from the CI log. The file has three backticked test names with spaces; one of them is `two offline edits of one note, newest wins`. Everything else (`runTest`, `kotlin.test`, `copy`) is ordinary multiplatform code.

PASS if the answer identifies the backticked test name that contains a comma (`two offline edits of one note, newest wins`) as the cause, and says that Kotlin/Native rejects `,` in a declaration name (an "illegal characters" error) while the JVM accepts it. That is why the JVM run stays green and only the iOS test compile fails.

FAIL if any of these is true:
- It says spaces in backticked names are not allowed on Kotlin/Native, or that backticked names are not supported there.
- It blames `runTest`, `kotlin.test`, `copy`, a missing iOS test dependency, or an expect/actual gap.
- It points at another test name or line.
- It lists the comma as one of several equally likely candidates without committing to it.
