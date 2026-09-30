---
name: design-iterate
description: "Use when polishing or reviewing a UI screen on iOS, Android or the web: capture before and after screenshots on the device or browser, review, fix, and capture again. Not for backend or non-visual tasks. Triggers - UI polish, design review, screenshot, spacing, alignment, contrast, touch target, dark mode, reduced motion."
license: MIT
metadata:
  author: Filip Kowalski
---

# Design Iterate

Verify the requested visual outcome on the actual screen. Fix the reported defect and any critical regression introduced by the change. Scores help prioritize improvements; they do not replace acceptance criteria.

## Scope and capture

- Infer the screens in scope from the request and existing context. Ask only about a material unresolved choice. Do not silently drop requested screens because of a fixed per-run limit.
- Use available approved capture tooling: browser tools, Playwright, or `agent-browser` for web; the available device-interaction tooling for iOS; `adb` or the project's documented tooling for Android. Tool choice does not change the evidence required.
- Keep full-screen before/after captures in the session scratch directory. Reproduce the actual reported state and user flow. Verify meaningful dark-mode or large-text variants when affected.
- Read design guidance only for the platform, component, or motion issue being changed. Reuse guidance already loaded and respect the project's existing design system.
- For motion, pair this loop with `emil-design-eng` and `apple-design` (Emil Kowalski, MIT, https://github.com/emilkowalski/skills) and, in a Compose Multiplatform app, `kmp-compose-motion`. For capture on the iOS Simulator and Android, see `kmp-agent-device-testing`.

## Focused changes

For a small spacing, text-fit, or visual bug fix, inspect the before/after captures directly. Check the exact reported path and affected layout, contrast, touch targets, safe areas, and interactions. No separate reviewer agent or numerical score is required.

Finish when the requested defect is fixed and the affected behavior has been verified. Report any unavailable device or unverified state accurately.

## Substantial changes and explicit design reviews

Use an independent reviewer when available and permitted. Give it the current screenshot, intended user task, relevant product constraints, and accessibility requirements. Ask for concrete findings in these areas:

- UX and flow clarity.
- Visual hierarchy, spacing, alignment, typography, and color.
- Accessibility: contrast, touch targets, readability, and affected large-text states.
- Unnecessary decoration or generic patterns that obscure the product.

Each finding should name the affected element and distinguish a critical defect from an optional improvement. If scoring is useful, use a consistent 1–10 scale and treat 8.5 as an advisory target. A score never excuses a known critical defect.

Fix critical issues and the improvements needed to satisfy the brief, then recapture affected screens. Request another review only when changes or unresolved findings justify it. Reuse existing evidence while its inputs remain unchanged. If independent review is unavailable, perform the same checks directly and report that limitation; do not stop solely because an agent type is unavailable.

Limit optional polish to four iterations or stop earlier when improvements plateau. If critical issues remain, continue the necessary fix or report a concrete blocker; reaching an iteration cap does not make the screen complete.

## Finish

Use a polish skill, if you have one, or perform these checks directly: text fit, visual alignment, contrast, focus states, loading/empty/error states, responsive or safe-area behavior, and reduced-motion behavior where affected. Not having a polish skill is not a blocker.

For motion changes, verify interruption, exit behavior, and reduced-motion settings. Respect `prefers-reduced-motion`, platform Reduce Motion, or `MediaQuery.disableAnimations` as applicable.

Report the result, relevant fixes, verification device/browser, and before/after screenshot links. Identify unresolved critical issues and unverified states; do not claim completion from a score alone.
