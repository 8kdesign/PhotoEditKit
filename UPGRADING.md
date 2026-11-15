# Upgrading PhotoEditKit

What to change in your app for each release, newest first.

## 1.2.5

### Matte is a theme

Matte is no longer a regular style. It is offered only with the themes —
`includeThemes: true`, or `EditController.offersThemeSuggestions` — after the
regular list and marked `isTheme`, like Manga, B&W and Film. It never comes
first, never counts toward `maxSuggestions`, and `topSuggestion` no longer
applies it.

- To keep offering Matte, turn the themes on. If you show themes apart from
  the regular looks (by `isTheme`), Matte now appears among them.
- `AILook.matte` still exists: `disabledLooks: [.matte]` still leaves it out.
- On Android (`aicore_suggest`), the same applies: send `"includeThemes": true`
  to get `"matte"`, which comes back with `"isTheme": true`.

## 1.2.1

### Add the Vibrance slider

There is a new setting, `EditSettingType.vibrance` (−1 … 1, untouched at 0).
The Vivid look now uses it, so show it in your editor:

- If you build your sliders from `availableEditSettings`, there is nothing to
  do. It appears after Saturation.
- If you list the settings yourself, add `.vibrance` after `.saturation`.

If you save edit values yourself, include `.vibrance` too.
