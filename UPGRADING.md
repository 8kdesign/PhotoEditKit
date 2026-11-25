# Upgrading PhotoEditKit

What to change in your app for each release, newest first.

## Unreleased

### 10-bit HEIC exports

`exportData` and `exportLivePhoto` with `type: .heic` now render at 10 bits a
channel and write a 10-bit HEIC, so strong lifts on skies and other smooth
gradients don't band. JPEG exports are unchanged (8-bit). The HDR gain map
and metadata are kept as before. A 10-bit HEIC is larger: about 2.4× the
8-bit one on the test photos. If you render with `renderForExport` and
encode yourself, pass `tenBit: true` for a HEIC.

### Original when nothing else is offered

The AI suggestions are never an empty list now: when no look is worth
offering, an **Original** suggestion (no edit) comes back on its own, with
or without `offerOriginal`. An empty list now means only that the pass was
cancelled or the photo couldn't be read. `EditController` reports `.done`
with that one suggestion where it reported `.unavailable`, so code that
showed a "no suggestions" state there will show the Original card instead.
On Android, `aicore_suggest` returns `"original"` in the same case.

### Export and `EditConfig` changes

- `EditConfig ==` now compares the crop and the quarter turns too, not just
  the slider values. Code that watches `currentConfig` for changes now sees
  a crop or a turn made on its own.
- `editRotation` is optional on `renderForExport`, `exportData`,
  `exportLivePhoto` and `exportLivePhotoVideo`, and defaults to
  `config.editRotation`, which `currentConfig` already carries. Existing
  calls that pass it still compile and behave the same.
- A save teaches the AI suggestions only once it succeeds: `exportData` and
  `exportLivePhoto` record it after encoding, and `renderForExport` after
  rendering. A failed save no longer teaches them.

### New Digicam theme

There is a new theme, Digicam: an early-2000s point-and-shoot look. Like the
other themes it is offered only with `includeThemes: true` (or
`EditController.offersThemeSuggestions`), after the regular list and marked
`isTheme`.

- `AILook` has a new case, `.digicam`. A `switch` over `AILook` that lists
  every case without a `default:` no longer compiles: handle `.digicam` or
  add a `default:`.
- To leave it out, pass `disabledLooks: [.digicam]`.
- It is never offered for a video: it is in `VideoLook.disabledLooks`, and a
  frame passed with `movieFrame: true` doesn't get it either.
- On Android (`aicore_suggest`), it comes back as `"digicam"` with
  `"isTheme": true`, and `"disabledLooks": ["digicam"]` leaves it out.

### New Lo-Fi and Noise settings

Two new sliders, both 0 … 1 and untouched at 0: `EditSettingType.lofi`
(an old low-resolution camera's softness and JPEG artefacts) and `EditSettingType.noise` (sensor
noise, mostly colour, mostly in the shadows). Both are in
`availableEditSettings`, between Sharpness, Noise Reduction and Vignette, so
an app that builds its slider list from it shows them with no change.

- A `switch` over `EditSettingType` that lists every case without a
  `default:` no longer compiles: handle `.lofi` and `.noise`.
- An app with its own fixed list of sliders doesn't show them until it adds
  them; Digicam still applies them either way.
- `.noise` is in `VideoLook.excludedSettings`: a video frame never gets it,
  so an app that hides those settings for a video hides Noise with no
  change. One with its own list should leave Noise out for videos.
- On Android, both are left out of `aicore_render_lut` like sharpness and
  noise reduction (they aren't per-pixel colour); a host that renders the
  edit itself needs its own version of each to show them.

### Highlights goes above zero

`EditSettingType.highlight` is now −1 … 1 (it was −1 … 0). Below zero it is
unchanged. Above zero it pushes the highlights up and clips them hard at
white, one colour channel at a time, like an old camera sensor; Digicam uses
+0.5.

- If you build your sliders from `EditSetting` (`minValue`/`maxValue`), the
  Highlights slider now reaches +1 with nothing to change. Its rest value is
  still 0, now in the middle: if you draw it as a one-sided slider, draw it as
  centred.
- If you check `.highlight` for "is this pulled", compare against 0 (or the
  filter value against 1) rather than assuming every non-zero value is a pull.
- On Android, `tone[2]` (highlight, neutral 1) passed to your
  `aicore_tone_patch_fn` can now be over 1. A host with its own renderer
  should apply the push (`HighlightPush` in `ColorGains.swift`) after the rest
  of the tone chain and pass at most 1 to its shadows and highlights filter.
  `aicore_render_lut` and AICore's own renderer already do it.

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
