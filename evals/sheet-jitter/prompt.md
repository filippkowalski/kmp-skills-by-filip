---
name: sheet-jitter
tags: [kmp-sheets-keyboard]
runs: 2
max_turns: 6
timeout_seconds: 300
allowed_tools: [Read, Glob, Grep, Skill]
description: "A ModalBottomSheet whose content is nearly as tall as the sheet's maximum re-solves its anchors every frame and never settles; the team suspects a recomposition loop."
expected_outcome: "Explains that the sheet sizes itself from content and loops when content sits within about a drag handle of the maximum height, and caps the content near 85% of the available height with a scrolling body."
---
Our filter sheet in a shop app (CMP 1.11.1, material3 1.9.0, Kotlin 2.4.10) moves up and down by a few dp every frame and never settles, on a Pixel 7 and an iPhone 15. On a Pixel 8 Pro, an iPad and a Pixel 6a it is perfectly still. A single screenshot looks fine; you only see it in a screen recording.

```kotlin
@Composable
fun FilterSheet(filters: Filters, onChange: (Filters) -> Unit, onApply: () -> Unit, onClose: () -> Unit) {
    ModalBottomSheet(
        onDismissRequest = onClose,
        sheetState = rememberModalBottomSheetState(skipPartiallyExpanded = true),
    ) {
        Column(Modifier.fillMaxWidth().padding(horizontal = 20.dp)) {
            Text(stringResource(Res.string.filters_title), style = MaterialTheme.typography.titleLarge)
            FilterGroup(Res.string.filter_brand, filters.brands, onChange)   // 4 rows
            FilterGroup(Res.string.filter_size, filters.sizes, onChange)     // 5 rows
            FilterGroup(Res.string.filter_color, filters.colors, onChange)   // 4 rows
            PriceSlider(filters.price, onChange)
            Button(onApply, Modifier.fillMaxWidth().padding(vertical = 16.dp)) {
                Text(stringResource(Res.string.filters_apply))
            }
        }
    }
}
```

When we removed the colour group, it stopped bouncing on the Pixel 7 too, so the team thinks `FilterGroup` has a recomposition loop. But Layout Inspector shows no recompositions inside the groups while the sheet bounces, and nothing in `filters` changes. We also tried `skipPartiallyExpanded = false`: same bounce. Where is the loop, and how do we make the sheet settle on every phone?
