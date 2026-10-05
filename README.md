# PhotoEditKit

Photo editing for iOS: a Core Image edit pipeline, a ready-made editor model
with undo/redo and export, and on-device AI that suggests edits for a photo.

Everything runs on the device. PhotoEditKit makes no network requests.

- [Requirements](#requirements)
- [Installation](#installation)
- [Quick start: let the AI edit a photo](#quick-start-let-the-ai-edit-a-photo)
- [Edits: `EditConfig`](#edits-editconfig)
- [Rendering an edit](#rendering-an-edit)
- [AI suggestions](#ai-suggestions)
- [Building an editor with `EditController`](#building-an-editor-with-editcontroller)
- [Exporting with HDR](#exporting-with-hdr)
- [Matching a style](#matching-a-style)
- [Threading](#threading)
- [Localization](#localization)

## Requirements

- iOS 17.6 or later, on a device or the simulator
- Xcode with the same Swift compiler that built this release, or a newer one
- Safe to use in app extensions

## Installation

PhotoEditKit ships as a binary `PhotoEditKit.xcframework`.

### Swift Package Manager

In Xcode, choose **File → Add Package Dependencies…**, enter the package URL
you were given and add the **PhotoEditKit** library to your app target.

To declare the dependency in your own `Package.swift`:

```swift
.package(url: "<package URL>", from: "1.0.0")
```

### Manual

1. Copy `PhotoEditKit.xcframework` into your project folder.
2. In your app target, go to **General → Frameworks, Libraries, and Embedded
   Content**, click **+**, then **Add Other… → Add Files…** and pick the
   `.xcframework`.
3. Set it to **Embed & Sign**. For an app extension that also uses it, add it
   to the extension as **Do Not Embed**; the app embeds it.

If the app crashes at launch with `Library not loaded: @rpath/PhotoEditKit.framework`,
the framework isn't embedded. Check step 3.

## Quick start: let the AI edit a photo

```swift
import PhotoEditKit

// Off the main thread: this measures the photo.
if let config = AIPostProcessingHelper.topSuggestion(for: cgImage, metadata: properties) {
    let edited: CIImage = CIImage(cgImage: cgImage).applyingEditConfig(config)
} else {
    // The photo is already good. Keep it as it is.
}
```

`metadata` is the photo's ImageIO properties
(`CGImageSourceCopyPropertiesAtIndex`). It's optional, but the suggestions are
better with it, because it tells the AI how the photo was shot.

## Edits: `EditConfig`

An `EditConfig` describes one complete edit: slider values, crop and rotation.
The AI produces them, the editor keeps a history of them, and the pipeline
renders them.

```swift
let config = EditConfig(properties: [
    .exposure: 0.2,
    .contrast: 0.1,
    .saturation: -0.3,
    .vignette: 0.4,
])
```

Each value is an **offset from neutral**, so `0` (or leaving the key out)
means "unchanged". To scale an edit up or down, multiply its values. Leave
`.aspectratio` and `.rotation` alone when you do, because half a crop isn't a
gentler crop:

```swift
var properties = config.properties
for (key, value) in properties where key != .aspectratio && key != .rotation {
    properties[key] = value * 0.5
}
let gentler = EditConfig(properties: properties, crop: config.crop, editRotation: config.editRotation)
```

### Settings and ranges

| `EditSettingType` | Range | |
|---|---|---|
| `.exposure` | −0.5 … 0.5 | |
| `.brightness` | −0.5 … 0.5 | |
| `.contrast` | −0.5 … 0.5 | |
| `.shadow` | −1 … 1 | positive lifts shadows |
| `.highlight` | −1 … 0 | negative recovers highlights |
| `.saturation` | −1 … 1 | |
| `.temperature` | −1 … 1 | positive is warmer |
| `.shadowWarmth` | −1 … 1 | warms or cools the shadows only |
| `.sharpness` | 0 … 2 | |
| `.noiseReduction` | 0 … 1 | |
| `.vignette` | −1 … 1 | |
| `.boostRed` `.boostYellow` `.boostGreen` `.boostCyan` `.boostBlue` `.boostMagenta` | −1 … 1 | per-colour saturation |
| `.rotation` | −45 … 45 | degrees, straightening |
| `.depth` | 0 … 40 | background blur; needs depth data |

The ranges are also available as constants (`EXPOSURE_MIN`, `EXPOSURE_MAX`, …).
`type.getSetting()` returns an `EditSetting` with the localized name, an SF
Symbol and the range, which is everything you need to build a slider.
`availableEditSettings`, `colorBoostSettings` and `rotationSettings` list the
settings in display order.

Crop is `config.crop`, a rect in normalized coordinates (`DEFAULT_CROP` is the
whole image). Quarter turns and flips are `config.editRotation` (`EditRotation`).

Several configs can be merged into one with `[config1, config2].combine()`.

## Rendering an edit

**Core Image.** Applies the colour and tone settings:

```swift
let output: CIImage = input.applyingEditConfig(config)
```

When you're rendering a preview that is smaller than the photo, pass
`renderScale` (preview size ÷ full size) so sharpening and noise reduction look
the same as they will at full size.

**Full render, including crop and rotation.** Use `PostProcessingHelper`. The
vignette is applied last, so it stays centred on the cropped photo:

```swift
// On LogicActor
let helper = PostProcessingHelper(image: cgImage)
helper.applyEditConfig(config: config, includeVignette: false)
helper.applyRotationCropConfig(config: config)
helper.applyFullRotateAndFlip(editRotation: config.editRotation)
helper.applyVignette(config: config)
let result: CGImage? = helper.toCGImage()
```

`PostProcessingHelper.renderedEdit(of:config:rawRecovery:)` is a shortcut for the
colour and tone part alone.

## AI suggestions

All AI is reached through `AIPostProcessingHelper`. The calls are synchronous
and take a noticeable amount of time on full-resolution photos, so run them off
the main thread (for example inside `runOnAIThread { … }`).

### One edit

```swift
AIPostProcessingHelper.topSuggestion(for: cgImage, metadata: properties) -> EditConfig?
```

Returns the edit the photo most needs, or `nil` when it doesn't need one. It
never crops or rotates. A copy of about 512 px on the long edge is enough to
pass in, and is much faster than full resolution. A `CGImage` carries no
orientation, so turn it upright before passing it in.

### A list of looks

```swift
let looks: [EditSuggestion] = AIPostProcessingHelper.suggestions(for: uiImage, metadata: properties)
for look in looks {
    look.title      // LocalizedStringResource, e.g. "Brighten"
    look.config     // EditConfig to apply
    look.thumbnail  // small rendered preview, ready for a picker
}
```

Returns up to ten looks, best first. Pass the photo at full resolution. Its
`imageOrientation` is honoured. If the
calling `Task` is cancelled, it stops early and returns an empty array.

If you develop RAW files yourself, pass what you know about the RAW for better
results:

```swift
let raw = AIPostProcessingHelper.RawContext(cameraReference: embeddedPreview, rawRecovery: nil)
AIPostProcessingHelper.suggestions(for: develop, raw: raw, metadata: properties)
```

### Scene

```swift
let scene: AIScene? = AIPostProcessingHelper.classifyScene(cgImage)
scene?.title   // localized name
scene?.icon    // SF Symbol name
```

The scenes are `people`, `animals`, `food`, `object`, `indoor`, `city`, `sky`,
`nature` and `light`. It returns `nil` when the photo couldn't be classified.

## Building an editor with `EditController`

`EditController` is an `ObservableObject` that runs a complete editing
session: it loads the photo, renders previews as edits change, keeps
undo/redo, runs the AI suggestions and renders the final export. You provide
the photo through an `EditSource` and build whatever UI you like on top.

### 1. Provide the photo

`EditController` doesn't know about the photo library or files. It reads
everything through your `EditSource`. For an ordinary image file, most methods
have nothing to return:

```swift
final class FileEditSource: EditSource {
    let data: Data
    let imageSource: CGImageSource

    init?(data: Data) {
        guard let source = CGImageSourceCreateWithData(data as CFData, nil) else { return nil }
        self.data = data
        self.imageSource = source
    }

    var isRaw: Bool { false }

    var orientation: CGImagePropertyOrientation {
        let properties = CGImageSourceCopyPropertiesAtIndex(imageSource, 0, nil) as? [String: Any]
        let value = properties?[kCGImagePropertyOrientation as String] as? UInt32
        return value.flatMap(CGImagePropertyOrientation.init(rawValue:)) ?? .up
    }

    // The full-resolution photo. A UIImage's imageOrientation is honoured, so
    // you don't need to redraw a photo that carries an EXIF orientation.
    @LogicActor func loadOriginal() async -> EditSourceImage? {
        let options: [CFString: Any] = [
            kCGImageSourceCreateThumbnailFromImageAlways: true,
            kCGImageSourceCreateThumbnailWithTransform: true,
        ]
        guard let image = CGImageSourceCreateThumbnailAtIndex(imageSource, 0, options as CFDictionary) else { return nil }
        return EditSourceImage(image: UIImage(cgImage: image), isDevelopedRaw: false)
    }

    // The ImageIO properties. The AI reads exposure information from them.
    func loadMetadata() async -> [String: Any]? {
        CGImageSourceCopyPropertiesAtIndex(imageSource, 0, nil) as? [String: Any]
    }

    // The encoded file, so an export can keep its HDR gain map.
    func loadFileData() async -> (data: Data, orientation: CGImagePropertyOrientation)? {
        (data, orientation)
    }

    // Only used for RAW files and photos with depth.
    @LogicActor func loadRawExtras(developSize: CGSize, exposures: [Float], referenceMaxDimension: CGFloat) async -> RawEditingExtras {
        RawEditingExtras(cameraReference: nil, offsetDevelops: [:])
    }
    func developRaw(targetSize: CGSize, exposures: [Float]) async -> [Float: CGImage] { [:] }
    @LogicActor func loadDepth() async -> (data: AVDepthData, orientation: Int32)? { nil }
    @LogicActor func defaultBlurRadius(depthData: AVDepthData, imageSize: CGSize) async -> CGFloat { 0 }
}
```

`loadPreview(maxDimension:)` is optional. Implement it if you can make a small
copy more cheaply than shrinking the original, for example if you already have
one on screen.

### 2. Run the session

```swift
// On the main actor, e.g. in your view model
let editor = EditController()

// Show `editor.editedImage` (a preview-sized UIImage) in your UI. It appears
// as soon as the photo has loaded, and updates after every edit.
// Observe `editor.aiProcessingState` for suggestions.

editor.processAI()                        // ask for suggestions; runs once the photo has loaded
Task { await editor.prepare(source: source) }
```

`aiProcessingState` moves from `.pending` to `.loading` to
`.done(result: [EditSuggestion])`, or to `.unavailable` when there are no
suggestions to offer.

### 3. Edit

The configs you pass to `previewConfig` and `addConfig` are **changes**, added
on top of the edit so far. `addConfig(EditConfig(properties: [.exposure: 0.1]))`
twice gives an exposure of 0.2.

| | |
|---|---|
| `applySuggestion(_:)` | apply an AI look; `nil` removes it. `appliedSuggestionID` tells you which one is on |
| `previewConfig(_:)` | show a change while a slider is moving, without adding it to the history. Each call replaces the previous preview |
| `confirmPreviewConfig()` | commit the previewed change when the slider is released |
| `addConfig(_:)` | commit a change in one step |
| `undoConfig()` / `redoConfig()` | with `canUndo()` / `canRedo()`; see below for keeping buttons up to date |
| `historyCount` | how many steps undo can go back. To stop undo at a point (e.g. right after a look was applied), read it there and allow undo only while it's larger |
| `hasEdits()` | whether anything has been changed |
| `showOriginalImage` | a flag for your UI, e.g. while a "compare" button is held. The controller doesn't swap images itself: show `sourcePreviewImage` while it's `true` |
| `currentConfig` | the edit as it stands |
| `aspectRatio`, `editRotation` | crop ratio and quarter turns |

#### Keeping undo and redo buttons up to date

Every edit, undo and redo fires the controller's `objectWillChange`. A SwiftUI
view that observes the controller re-reads `canUndo()` and `canRedo()`
automatically:

```swift
@ObservedObject var editor: EditController

Button("Undo") { editor.undoConfig() }.disabled(!editor.canUndo())
```

In UIKit, subscribe to `objectWillChange` and read the values on the next turn
of the main run loop. `objectWillChange` fires just before the change, so
reading them inside the sink gives the old values:

```swift
cancellable = editor.objectWillChange
    .receive(on: RunLoop.main)
    .sink { [weak self] in
        guard let self else { return }
        undoButton.isEnabled = editor.canUndo()
        redoButton.isEnabled = editor.canRedo()
    }
```

### 4. Export and finish

```swift
let config = editor.currentConfig
let rotation = editor.editRotation
if let rendered = await editor.renderForExport(config: config, editRotation: rotation) {
    rendered.image     // full-resolution CGImage
    rendered.gainMap   // HDR gain map, when the original had one
}
editor.destroy()       // when the editing session ends
```

## Exporting with HDR

Photos taken on recent iPhones carry an HDR gain map. To keep it in an export,
encode the image and the gain map together:

```swift
let data = rendered.gainMap.flatMap {
    HDRGainMap.encode(rendered.image, gainMap: $0, type: .heic, quality: 0.9)
} ?? rendered.image.convertCGImage(to: .heic, quality: 0.9)
```

If you then copy the data into a `CGImageDestination` to add metadata, call
`HDRGainMap.copy(from:to:)` as well, because ImageIO leaves the gain map behind
when it copies an image. `HDRGainMap.read(from:orientation:)` and
`hasGainMap(_:)` read and detect gain maps.

## Matching a style

`StyleMatcher` makes one photo's tones match a reference photo, or the average
of several:

```swift
// On LogicActor
guard let reference = StyleMatcher.statistics(for: referenceImage),
      let target = StyleMatcher.statistics(for: photo),
      let transform = StyleMatcher.transform(for: target, matching: reference),
      transform.isMeaningful else { return }
let matched = photo.applyingStyleTransform(transform)
```

Use `StyleMatcher.average(of:)` to combine several references into one.

## Threading

PhotoEditKit uses three global actors of its own. Calls marked with them can
be awaited from anywhere:

- `LogicActor` for loading, rendering and export (`EditController.prepare`,
  `renderForExport`, `PostProcessingHelper` geometry, `StyleMatcher`).
- `AIActor` for AI work. `runOnAIThread { … }` runs a closure on it.
  `EditController` uses it for suggestions.
- `MainActor` for everything on `EditController` that changes the edit
  (`addConfig`, `applySuggestion`, `undoConfig`, …).

`AIPostProcessingHelper` and `applyingEditConfig` aren't tied to an actor. Call
them from wherever suits you, just not on the main thread for full-size images.

## Localization

Setting names, look titles and scene names come back as
`LocalizedStringResource` in English and Simplified Chinese, and follow the
device language. Display them with `Text(title)` in SwiftUI, or
`String(localized: title)` in UIKit.
