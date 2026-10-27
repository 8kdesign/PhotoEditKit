# Upgrading PhotoEditKit

What to change in your app for each release, newest first.

## 1.2.1

### Add the Vibrance slider

There is a new setting, `EditSettingType.vibrance` (−1 … 1, untouched at 0).
The Vivid look now uses it, so show it in your editor:

- If you build your sliders from `availableEditSettings`, there is nothing to
  do. It appears after Saturation.
- If you list the settings yourself, add `.vibrance` after `.saturation`.

If you save edit values yourself, include `.vibrance` too.
