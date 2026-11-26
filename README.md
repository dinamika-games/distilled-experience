# How We Build Dinamika Games

This is the guideline for every new Dinamika game (see the [Manifesto](../MANIFESTO.md)): a game played with the body in front of a webcam, by children and their families, together, offline.
It is the crystallized experience of the games we shipped so far.
Each section says **what to do**, gives **the code to copy**, and says **why** in a line where the reason was learned the hard way.

Do it this way unless the game has a reason not to; if it has one, write the reason into its DESIGN.md.

**Contents**

1. [The rules on one page](#1-the-rules-on-one-page)
   * [Prototype first, then tests and translations](#prototype-first-then-tests-and-translations)
2. [Stack and repository layout](#2-stack-and-repository-layout)
3. [Git LFS](#3-git-lfs)
4. [Code structure](#4-code-structure)
5. [The camera on Windows, macOS and Linux](#5-the-camera-on-windows-macos-and-linux)
6. [Vision](#6-vision)
7. [Smoothing: the 1€ filter](#7-smoothing-the-1-filter)
8. [The hand cursor and the dwell buttons](#8-the-hand-cursor-and-the-dwell-buttons)
9. [Button selection with shaders](#9-button-selection-with-shaders)
10. [The player is always visible](#10-the-player-is-always-visible)
11. [Translations with PO and POT](#11-translations-with-po-and-pot)
12. [Testing](#12-testing)
13. [Saves](#13-saves)
14. [Pictures: generative AI on SVG sketches](#14-pictures-generative-ai-on-svg-sketches)
15. [Sound effects and music](#15-sound-effects-and-music)
16. [Export and `dev/publish.py`](#16-export-and-devpublishpy)
17. [GitHub Actions](#17-github-actions)
18. [Development scripts in `dev/`](#18-development-scripts-in-dev)
19. [README, CONTRIBUTING, DESIGN, THIRD_PARTY_NOTICES](#19-readme-contributing-design-third_party_notices)
20. [Game design for bodies](#20-game-design-for-bodies)
21. [Godot gotchas](#21-godot-gotchas)
22. [Checklist for a new game](#22-checklist-for-a-new-game)

---

## 1. The rules on one page

**Platform**
* Godot 4.6.2 .NET, .NET 8, **GL Compatibility** renderer, 1920x1080, `canvas_items` stretch, fullscreen.
  (Pixel art: 960x540, integer scale.)
* Exports for Windows x86_64, Linux x86_64 and macOS **universal** (arm64 + x86_64), built on Linux in CI.
* **ONNX Runtime 1.19.2**, CPU, pinned: the last version whose macOS libraries run on macOS 11 on Apple Silicon and Intel.
* Windows camera through **CameraServerExtension**, loaded at runtime.

**Vision**
* **RTMO-t detects** the bodies, only when a player must be (re-)found.
  **RTMPose-t tracks** each found body every frame.
* **YOLOX Body-Head-Hand** finds the hands and heads; **the cursor follows the hand box, never the wrist.**
* The **1€ filter** smooths, with speeds in **body units**.
  Every threshold is in body units.
* No network.
  ONNX Runtime telemetry off.
  Pictures are never saved, except in consented recordings that are never committed.

**Code**
* **Classes, not positional arrays.**
  No dictionaries as records.
* The vision and the rules are **Godot-free** and unit-tested; one bridge node connects them to the game.
* Game logic and UI run on **game time**.

**Order of work**
* **Prototype first**: until the game works, only the formatting, lint and typing checks (`precommit.py --prototype`).
* **Then** the tests and the translations, before the first release (see below).

**Quality**
* Unit tests, **property-based tests** (CsCheck), and a **seeded end-to-end bot** that plays the whole game through the bridge at high speed and checks **contracts** on every frame:
  * **no text overflows**, no GUI element overflows the screen, its parent or another element;
  * **the player is never covered by a GUI/HUD element**;
  * every text is in the language of the game; the rules of the game hold.
* Any Godot `ERROR` line fails a test.

**UI**
* The dim mirrored camera picture behind everything; avatars drawn exactly on the players.
* Dwell buttons with a **shader** glow and a progress ring; sticky selection; at most 4–6 big buttons per screen.
* All texts through `tr()`, translated with gettext **PO/POT**.

**Assets**
* Pictures: **SVG sketch → generative AI → cut by script**, in separate **flat** folders; overlays sketched with **gray shapes for the player and dots for the anchors**.
  Prompts and raw paintings are committed.
* Sound effects: **CC0 or CC BY 4.0**, license checked by script.
  Music: **MIDI rendered with FluidSynth**.
* Binaries in **Git LFS**.

**Delivery**
* Every development script is **Python in `dev/`**; `dev/publish.py` exports, checks and zips every platform.
* GitHub Actions run CI on every push and publish on every release.
* README (players), CONTRIBUTING (developers), DESIGN (designers), THIRD_PARTY_NOTICES (licenses).

### Prototype first, then tests and translations

**While prototyping**, until the game is playable end to end with real players, run only the fast checks:

```
venv/bin/python dev/continuous_integration/precommit.py --prototype [--overwrite]
```

* formatting (gdformat, `dotnet format`, black) and lint (gdlint);
* typing: `dotnet build` and `--check-only` on every GDScript (static types everywhere, 4.4);
* no unit tests, no property-based tests, no end-to-end bot, no contracts, no balance bots;
  no `languages.json`, PO/POT, `Grammar` or translation checks;
* CI runs the same: `precommit.py --prototype` in the second step of `ci.yaml` (17).

Write the texts already as whole sentences with named placeholders in `tr("...")`: it costs nothing now and makes the translation later mechanical.

**Once the game works** (a play test with children shows the mechanics hold), harden it before the first release, in this order:

1. the unit and property-based tests of the Godot-free core (12.1, 12.2) and the contracts in the code (12.3);
2. the fake vision, the end-to-end bot and its contracts (12.4, 12.5), the test scenes (12.6), the balance bots (12.7);
3. the translations (11): `languages.json`, `# TRANSLATORS:` comments, `Grammar`, extraction and checks;
4. drop `--prototype` in `ci.yaml`: from now on every step runs on every push.

Why: tests and translations of mechanics that are still changing are thrown away with them, and slow every iteration of the prototype; the fast checks still catch the typos and type errors at once.

---

## 2. Stack and repository layout

```
<game>/
├── project.godot            # config/version is the single source of the version
├── <Game>.csproj / .sln
├── export_presets.cfg       # Windows, Linux, macOS (universal)
├── README.md  CONTRIBUTING.md  DESIGN.md  THIRD_PARTY_NOTICES.md  LICENSE
├── .gitattributes  .gitignore  gdformatrc  .editorconfig
├── .config/dotnet-tools.json            # dotnet-format pinned
├── addons/CameraServerExtension/        # + .gdignore (loaded by hand on Windows)
├── assets/                              # only what the game loads; flat folders
│   ├── pictures/                        # <sheet>_<item>.png + <sheet>.json (anchors)
│   ├── sounds/                          # <name>.ogg
│   ├── music/                           # <slug>.mid + <slug>.ogg
│   └── fonts/                           # OFL fonts covering every language
├── data/                                # JSON content, loaded into typed classes at start
├── scenes/
│   ├── vision/
│   │   ├── core/*.cs                    # Godot-free
│   │   ├── rtmo/model/                  # rtmo.onnx + rtmo.json + LICENSE
│   │   ├── rtmpose/model/
│   │   ├── yolox/model/
│   │   └── vision_service/VisionService.cs   # the bridge, autoload "Vision"
│   ├── languages/                       # languages.gd, grammar.gd
│   ├── ui/                              # hand cursor, dwell button, hud layout, shaders
│   ├── tests/                           # test scenes per mechanic (excluded from exports)
│   └── ...                              # game scenes; rules in Godot-free classes
├── translations/                        # messages.pot, <lang>.po, languages.json
├── licenses/                            # + .gdignore; godot/ generated, the rest copied at packaging
├── tests/
│   ├── <Game>.Tests/                    # xUnit + CsCheck; links scenes/**/core/*.cs
│   └── godot/                           # headless tests; end2end/ (bot, fake vision, contracts)
├── dev/                                 # + .gdignore; never shipped
│   ├── requirements.txt
│   ├── continuous_integration/precommit.py
│   ├── publish.py
│   ├── write_godot_licenses.gd
│   ├── translations/{extract,check}.py
│   ├── pictures/{sketches,prompts,painted}/  + cut_*.py
│   ├── sounds/{sources.json,make_sounds.py}
│   └── music/{sources.json,make_music.py}
├── .github/workflows/{ci,publish}.yaml
└── venv/                                # + .gdignore, git-ignored
```

**Put a `.gdignore` into every folder Godot must not import**: `dev/`, `tests/`, `licenses/`, `venv/`, `build/`, `dist/`, `addons/CameraServerExtension/`, and any agent worktree folder (e.g. `.claude/worktrees/`).
*Why:* otherwise Godot imports the venv, and tries to load the Windows DLL of a package in `build/` on every platform.

`<Game>.csproj`:

```xml
<Project Sdk="Godot.NET.Sdk/4.6.2">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <EnableDynamicLoading>true</EnableDynamicLoading>
    <Nullable>enable</Nullable>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    <!-- The camera frames are large objects. -->
    <ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
  </PropertyGroup>
  <ItemGroup>
    <!-- 1.19.x is the last version that runs on macOS 11 on arm64 and x86_64
         (1.20+ needs macOS 13.3, 1.24+ has no Intel build). The tests use the same version. -->
    <PackageReference Include="Microsoft.ML.OnnxRuntime" Version="1.19.2" />
  </ItemGroup>
  <ItemGroup>
    <Compile Remove="tests/**" />
    <Compile Remove="dev/**" />
    <Compile Remove="venv/**" />
  </ItemGroup>
</Project>
```

`project.godot` (the essentials):

```ini
[application]
config/name="<Game>"
config/version="1.0.0"
run/main_scene="res://scenes/main/main.tscn"

[autoload]
Vision="*res://scenes/vision/vision.tscn"
Languages="*res://scenes/languages/languages.gd"

[audio]
general/text_to_speech=true

[debug]
gdscript/warnings/untyped_declaration=2
gdscript/warnings/unsafe_property_access=1
gdscript/warnings/unsafe_method_access=1
gdscript/warnings/unsafe_call_argument=1

[display]
window/size/viewport_width=1920
window/size/viewport_height=1080
window/size/mode=4
window/stretch/mode="canvas_items"
window/stretch/aspect="keep"

[internationalization]
locale/fallback="en"

[rendering]
renderer/rendering_method="gl_compatibility"
renderer/rendering_method.mobile="gl_compatibility"
```

*Why GL Compatibility:* our games are 2D and must run on old school laptops (Intel HD 520 class) and on macOS 11 Intel; Compatibility runs wherever OpenGL 3.3 does.
Heavy shaders are rendered once into a texture (section 21).

---

## 3. Git LFS

**Every binary that is not tiny goes into LFS; text never does.**
Set it up in the first commit, before the first binary: moving files into LFS later leaves the old blobs in the history forever.

| Into LFS | Plain git | Never committed |
|---|---|---|
| `*.onnx` (models) | `*.svg` (sketches) | camera recordings and frames |
| `*.png *.jpg *.jpeg *.webp *.gif` | `*.mid` (a few KB, the source of the music) | `build/ dist/ *.zip` (packages go to the releases) |
| `*.ogg *.wav *.mp3` | `*.gd *.cs *.gdshader *.tscn *.tres *.import *.json *.csv *.md *.po *.pot` | `.godot/ venv/ .cache/ bin/ obj/` |
| `*.glb *.blend` | `*.abc` | soundfonts (installed, not vendored) |
| `*.ttf *.otf` | | |
| `*.dll *.so *.dylib` | | |

`.gitattributes`:

```
* text=auto eol=lf

*.onnx  filter=lfs diff=lfs merge=lfs -text
*.png   filter=lfs diff=lfs merge=lfs -text
*.jpg   filter=lfs diff=lfs merge=lfs -text
*.jpeg  filter=lfs diff=lfs merge=lfs -text
*.webp  filter=lfs diff=lfs merge=lfs -text
*.gif   filter=lfs diff=lfs merge=lfs -text
*.ogg   filter=lfs diff=lfs merge=lfs -text
*.wav   filter=lfs diff=lfs merge=lfs -text
*.mp3   filter=lfs diff=lfs merge=lfs -text
*.glb   filter=lfs diff=lfs merge=lfs -text
*.blend filter=lfs diff=lfs merge=lfs -text
*.ttf   filter=lfs diff=lfs merge=lfs -text
*.otf   filter=lfs diff=lfs merge=lfs -text
*.dll   filter=lfs diff=lfs merge=lfs -text
*.so    filter=lfs diff=lfs merge=lfs -text
*.dylib filter=lfs diff=lfs merge=lfs -text
```

`.gitignore`:

```
.godot/
venv/
venv-ml/
build/
dist/
.cache/
bin/
obj/
*.translation
/recordings/
```

* `git lfs install` once per machine, `git lfs pull` after cloning; both are in the setup block of CONTRIBUTING.
* **Enforce it**: `precommit.py` runs `git lfs fsck --pointers`, which fails on a file that matches a pattern but was committed as a plain blob, and a check that no binary extension is missing from `.gitattributes` (section 18).
  *Why:* rules that are only written get broken (JPEGs ended up in plain git, a typo `*.onx` went unnoticed).
* **Detect a missing `git lfs pull`.**
  The `.onnx` is then a 130-byte text pointer, which `FileAccess.FileExists` happily accepts:
  ```csharp
  private static bool IsLfsPointer(byte[] bytes) =>
      bytes.Length < 1024
      && System.Text.Encoding.ASCII.GetString(bytes).StartsWith("version https://git-lfs.github.com/spec/");
  // → status ModelsMissing: "The models are missing; run git lfs pull."
  ```
* Rendered outputs (sprites, Ogg) are committed although scripts make them: the game builds without the asset toolchain.
* CI downloads all LFS files on every run and counts against the free quota; cache `.git/lfs` (section 17).

---

## 4. Code structure

### 4.1 Godot-free core, one bridge

* `scenes/vision/core/*.cs`: frames, model wrappers, trackers, meters, filters.
  **No `using Godot;`.**
  Every file says so:
  ```csharp
  // NOTE (mristin):
  // The files in core/ must not depend on Godot so that we can unit-test the
  // vision outside of the engine. See tests/<Game>.Tests.
  ```
* `scenes/vision/vision_service/VisionService.cs`: the only node that touches the camera and the threads, an autoload `Vision`.
  It exposes **plain getters** in normalized, mirrored coordinates (`GetStatus()`, `HasPlayer(p)`, `GetHands()`, `GetKeypoints(p)`, `GetPlayerRects()`, the game's signals such as `GetStrokes(p)`), and queued events (`TakeEvents()`) for what must not be lost.
* The rules of the game are classes with `step(dt)` and no drawing, so bots can play them headless.
* The game never reads the vision any other way.
  *Why:* the end-to-end test replaces the bridge with a fake of the same method names (section 12.4), and a mouse/keyboard controller replaces it for development (`--no-camera`).

### 4.2 Classes, not positional arrays

**Values of different kinds that belong together get a class with named fields.**
Arrays are for values of the same kind (the 17 key points, the players).
Dictionaries are for maps, never for records.
*Why:* struct fields reused for other meanings ("the side travels as its rise") and string-keyed dictionaries (`encounter["state"]`) made code unreadable and broke only at runtime.

```gdscript
# GDScript: an inner class or a class_name file instead of [gap, at, heading] or a Dictionary.
class Sighting:
	extends RefCounted
	var gap: float
	var at: Vector2
	var heading: float

	func _init(a_gap: float, an_at: Vector2, a_heading: float) -> void:
		gap = a_gap
		at = an_at
		heading = a_heading
```

```csharp
// C#: one record per event kind.
public abstract record VisionEvent(int PlayerId, double Time);
public sealed record Jumped(int PlayerId, double Time, float Strength) : VisionEvent(PlayerId, Time);
public sealed record Swung(int PlayerId, double Time, Side Side) : VisionEvent(PlayerId, Time);
```

```python
# Python (dev/): frozen pydantic models or dataclasses.
class Sound(pydantic.BaseModel):
    model_config = pydantic.ConfigDict(frozen=True, extra="forbid")
    name: str
    start: float = 0.0
    duration: Optional[float] = None
```

Content in `data/*.json` is parsed **once, at start, into typed classes** and validated there (unknown fields, missing fields, references); the game never indexes the JSON.
The only exception to the rule: hot inner loops of the vision (input tensors; key points as `float[]` indexed by named constants such as `Pose.LeftWrist`).

### 4.3 Time

* **Game logic, UI, dwell, cursor animations: game time** (the accumulated `delta`), never `Time.get_ticks_msec()`.
  *Why:* a cursor on the wall clock broke when the end-to-end test sped up the game.
* **The vision: the timestamps of the frames.**
  With a camera they are wall time (the frames come in real time); with `--no-camera` and in tests the bridge uses game time:
  ```csharp
  private double Now => UseCamera ? _clock.Elapsed.TotalSeconds : _gameTime;
  ```

### 4.4 Typing

Typed GDScript is enforced by `project.godot` (`untyped_declaration=2`, section 2); C# has `Nullable`, analyzers and warnings as errors.
Doc comments state units: pixels, normalized, body units, seconds, degrees.

---

## 5. The camera on Windows, macOS and Linux

| OS | How | Must handle |
|---|---|---|
| Linux | `CameraServer` (V4L2) | Webcams halve their frame rate in dim light: switch off *exposure auto priority* (5.5). |
| macOS | `CameraServer` | Two planes (Y + CbCr): convert yourself (5.3). Feed stays off after the first permission prompt: retry (5.3). Entitlement and usage string (5.6). No formats listed. |
| Windows | [CameraServerExtension](https://github.com/j20001970/CameraServerExtension) (MIT, prebuilt DLL) | Godot 4.6 has no Windows camera (drop the extension once godotengine/godot#108538 lands). Load at runtime, copy next to the `.exe`. Virtual cameras (OBS) do not work. |

Ask for **640x480 YUYV** (V4L2 calls it `YUYV`, Media Foundation `YUY2`).
Wait for the feeds up to 5 s, and 1 s more after the first appears.
`--camera=<n>` picks one; `--no-camera` falls back to keyboard and mouse.
Tell the player why there is no camera: unsupported, denied ("allow it in Settings, Privacy & security, Camera"), none found, no 640x480, models missing.

### 5.1 The Windows extension

`addons/CameraServerExtension/` holds `CameraServerExtension.gdextension`, `libcameraserver-extension.windows.dll`, `LICENSE` and a **`.gdignore`** (Godot errors on every platform the `.gdextension` lists no library for).

```csharp
CameraServer.MonitoringFeeds = true;
if (OS.GetName() == "Windows")
{
    StartCameraExtension();
}

private void StartCameraExtension()
{
    // In the editor the addon folder; in an export next to the .exe (dev/publish.py copies it there).
    string directory = OS.HasFeature("editor")
        ? ProjectSettings.GlobalizePath("res://addons/CameraServerExtension")
        : OS.GetExecutablePath().GetBaseDir();
    string path = directory.PathJoin("CameraServerExtension.gdextension");
    if (!FileAccess.FileExists(path))
    {
        _noCameraReason = "unsupported";
        return;
    }
    var status = GDExtensionManager.LoadExtension(path);
    if ((status != GDExtensionManager.LoadStatus.Ok && status != GDExtensionManager.LoadStatus.AlreadyLoaded)
        || !ClassDB.ClassExists("CameraServerExtension"))
    {
        GD.PrintErr($"Could not load the camera extension from {path}: {status}");
        _noCameraReason = "unsupported";
        return;
    }

    // Keep the instance for the whole game: freeing it removes its feeds. Free it last on exit.
    _cameraExtension = ClassDB.Instantiate("CameraServerExtension").AsGodotObject();
    if (_cameraExtension != null && !_cameraExtension.Call("permission_granted").AsBool())
    {
        _noCameraReason = "denied";
    }
}
```

**Call the format methods by name, for every feed**: the methods of `CameraFeed` do not reach the extension's feeds, and a call by name works for all of them.

```csharp
private const int NoMatch = -1;
private const int AnyFormat = -2;   // macOS lists no formats: take what the system gives

private static Godot.Collections.Array GetFormats(CameraFeed feed) => feed.Call("get_formats").AsGodotArray();

private static void SetFormat(CameraFeed feed, int index) =>
    feed.Call("set_format", index, new Godot.Collections.Dictionary());

private static int FindFormat(CameraFeed feed)
{
    var formats = GetFormats(feed);
    if (formats.Count == 0) return AnyFormat;
    for (int i = 0; i < formats.Count; i++)
    {
        var format = formats[i].AsGodotDictionary();
        if (format["width"].AsInt32() != 640 || format["height"].AsInt32() != 480) continue;
        string pixelFormat = format.ContainsKey("format") ? format["format"].AsString() : "";
        // V4L2 on Linux calls the format YUYV, Media Foundation on Windows YUY2.
        if (pixelFormat.StartsWith("YUYV") || pixelFormat.StartsWith("YUY2")) return i;
    }
    return NoMatch;
}
```

### 5.2 Starting the feed and copying frames

```csharp
_feed.FeedIsActive = true;
_nextActivationAttempt = Now + ActivationRetryPeriod;   // 3 s
KeepFullFrameRate();                                    // Linux, 5.5

CameraTexture = new CameraTexture { CameraFeedId = _feed.GetId(), CameraIsActive = true };
_cbcrTexture = new CameraTexture
{
    CameraFeedId = _feed.GetId(),
    WhichFeed = CameraServer.FeedImage.CbcrImage,   // the second plane on macOS
    CameraIsActive = true,
};
_feed.FrameChanged += OnFrameChanged;
```

```csharp
private void OnFrameChanged()   // the main thread
{
    if (!_fullFrameRateKept)
    {
        // Once more with the first frame: a driver may reset the controls when the camera starts to stream.
        _fullFrameRateKept = true;
        KeepFullFrameRate();
    }

    bool forPose = _poseSlot.Requested;
    bool forHands = _handTracking && _handSlot.Requested;
    bool twoPlanes = _feed!.GetDatatype() == CameraFeed.FeedDataType.YcbcrSep;
    if (!forPose && !forHands && !twoPlanes) return;

    byte[] rgb; int width, height;
    if (twoPlanes)
    {
        Image? luma = CameraTexture!.GetImage();
        Image? chroma = _cbcrTexture?.GetImage();
        if (luma == null || luma.IsEmpty() || chroma == null || chroma.IsEmpty()) return;
        if (luma.GetFormat() != Image.Format.R8) luma.Convert(Image.Format.R8);
        if (chroma.GetFormat() != Image.Format.Rg8) chroma.Convert(Image.Format.Rg8);
        (rgb, width, height) = YcbcrConverter.ToRgb(
            luma.GetData(), luma.GetWidth(), luma.GetHeight(),
            chroma.GetData(), chroma.GetWidth(), chroma.GetHeight(), maxWidth: 640);
        UpdatePreview(rgb, width, height);   // a two-plane feed has no RGB texture to show
        if (!forPose && !forHands) return;
    }
    else
    {
        Image? image = CameraTexture!.GetImage();
        if (image == null || image.IsEmpty()) return;
        if (image.GetFormat() != Image.Format.Rgb8) image.Convert(Image.Format.Rgb8);
        rgb = image.GetData(); width = image.GetWidth(); height = image.GetHeight();
    }

    // We copy the raw bytes on the main thread so that the workers never touch engine objects.
    var frame = new Frame(rgb, width, height, Now);
    if (forPose) _poseSlot.Put(frame);
    if (forHands) _handSlot.Put(frame);
}
```

### 5.3 macOS: two planes and the permission

```csharp
/// The feed asks for the permission on the first start and stays off even when granted.
private void RetryActivation(double now)   // from _Process
{
    if (_feed == null || _feed.FeedIsActive || now < _nextActivationAttempt) return;
    _nextActivationAttempt = now + ActivationRetryPeriod;
    _feed.FeedIsActive = true;
    if (CameraTexture != null) CameraTexture.CameraIsActive = true;
    if (_cbcrTexture != null) _cbcrTexture.CameraIsActive = true;
}
```

`core/YcbcrConverter.cs` (full-range BT.601, downscaled to at most `maxWidth`, unit-tested):

```csharp
public static (byte[] Rgb, int Width, int Height) ToRgb(
    byte[] y, int lumaWidth, int lumaHeight,
    byte[] cbcr, int chromaWidth, int chromaHeight,
    int maxWidth)
{
    if (lumaWidth <= 0 || lumaHeight <= 0 || chromaWidth <= 0 || chromaHeight <= 0 || maxWidth <= 0)
        throw new ArgumentException($"Expected positive sizes, but got the luma {lumaWidth}x{lumaHeight}, " +
            $"the chroma {chromaWidth}x{chromaHeight} and the maximum width {maxWidth}.");
    if (y.Length < lumaWidth * lumaHeight || cbcr.Length < chromaWidth * chromaHeight * 2)
        throw new ArgumentException($"Expected at least {lumaWidth * lumaHeight} luma and " +
            $"{chromaWidth * chromaHeight * 2} chroma bytes, but got {y.Length} and {cbcr.Length}.");

    int step = Math.Max(1, (lumaWidth + maxWidth - 1) / maxWidth);
    int width = Math.Max(1, lumaWidth / step);
    int height = Math.Max(1, lumaHeight / step);
    var rgb = new byte[width * height * 3];
    for (int row = 0; row < height; row++)
    {
        int lumaRow = row * step;
        int chromaRow = Math.Min(chromaHeight - 1, lumaRow * chromaHeight / lumaHeight);
        for (int col = 0; col < width; col++)
        {
            int lumaCol = col * step;
            int chromaCol = Math.Min(chromaWidth - 1, lumaCol * chromaWidth / lumaWidth);
            float l = y[lumaRow * lumaWidth + lumaCol];
            int c = (chromaRow * chromaWidth + chromaCol) * 2;
            float cb = cbcr[c] - 128f;
            float cr = cbcr[c + 1] - 128f;
            int o = (row * width + col) * 3;
            rgb[o] = ClampByte(l + 1.402f * cr);
            rgb[o + 1] = ClampByte(l - 0.344136f * cb - 0.714136f * cr);
            rgb[o + 2] = ClampByte(l + 1.772f * cb);
        }
    }
    return (rgb, width, height);
}

private static byte ClampByte(float value) => (byte)Math.Clamp((int)MathF.Round(value), 0, 255);
```

### 5.4 Threads: request a frame, never queue frames

Each worker (pose, hands) is a background thread at `ThreadPriority.BelowNormal` that asks for the next frame through a one-frame slot: **frames never queue up, latency never builds**.
Results go back as immutable snapshots (`Volatile.Write`/`Volatile.Read`); status changes through `Callable.From(...).CallDeferred()`; events that must not be lost (a jump) through a `ConcurrentQueue`.
**Drop frames, never events.**

```csharp
private sealed class FrameSlot
{
    public volatile bool Requested;
    public readonly AutoResetEvent Ready = new(false);
    private readonly object _lock = new();
    private Frame? _pending;

    public void Put(Frame frame)
    {
        lock (_lock) { _pending = frame; }
        Requested = false;
        Ready.Set();
    }

    public Frame? Take()
    {
        lock (_lock) { var frame = _pending; _pending = null; return frame; }
    }
}

private Frame? NextFrame(FrameSlot slot)
{
    slot.Requested = true;
    if (!slot.Ready.WaitOne(500) || !_running) return null;
    return slot.Take();
}

private void PoseLoop()
{
    while (_running)
    {
        Frame? frame = NextFrame(_poseSlot);
        if (frame == null) continue;
        PoseSnapshot snapshot = _follower.Update(frame);   // Godot-free
        Volatile.Write(ref _latestPoses, snapshot);
    }
}
```

**Shut down** on `NotificationWMCloseRequest` and in `_ExitTree`: `_running = false`, join each thread **with a 2 s timeout**, deactivate the feed, free the camera extension last.

### 5.5 Linux: keep the full frame rate

In a dim living room most webcams drop from 30 to 15 fps.
Switch it off with V4L2; never throw:

```csharp
public static class LinuxCameraControls
{
    private const int O_RDWR = 2;
    private const ulong VIDIOC_S_CTRL = 0xC008561C;                 // _IOWR('V', 28, struct v4l2_control)
    private const uint V4L2_CID_EXPOSURE_AUTO_PRIORITY = 0x009A0903;

    [StructLayout(LayoutKind.Sequential)]
    private struct V4l2Control { public uint Id; public int Value; }

    [DllImport("libc", SetLastError = true)] private static extern int open(string path, int flags);
    [DllImport("libc", SetLastError = true)] private static extern int ioctl(int fd, ulong request, ref V4l2Control control);
    [DllImport("libc")] private static extern int close(int fd);

    /// The /dev/videoN devices whose name (as Godot shows the feed) matches.
    public static IEnumerable<string> DevicesNamed(string feedName, string sysfs = "/sys/class/video4linux")
    {
        if (!Directory.Exists(sysfs)) yield break;
        foreach (string dir in Directory.GetDirectories(sysfs).OrderBy(d => d))
        {
            string nameFile = Path.Combine(dir, "name");
            if (File.Exists(nameFile) && File.ReadAllText(nameFile).Trim() == feedName.Trim())
                yield return "/dev/" + Path.GetFileName(dir);
        }
    }

    public static bool KeepFullFrameRate(string device)
    {
        try
        {
            int fd = open(device, O_RDWR);
            if (fd < 0) return false;
            try
            {
                var control = new V4l2Control { Id = V4L2_CID_EXPOSURE_AUTO_PRIORITY, Value = 0 };
                return ioctl(fd, VIDIOC_S_CTRL, ref control) == 0;
            }
            finally { close(fd); }
        }
        catch (Exception) { return false; }   // the game works without it
    }
}
```

Unit-test `DevicesNamed` against a fake sysfs folder.

### 5.6 macOS export settings

```ini
[preset.2.options]
binary_format/architecture="universal"
application/bundle_identifier="io.github.dinamika-games.<game>"
application/min_macos_version_x86_64="10.15"
application/min_macos_version_arm64="11.00"
codesign/codesign=1                    ; ad hoc; players right-click → Open the first time
codesign/entitlements/camera=true
notarization/notarization=0
privacy/camera_usage_description="<Game> watches the players in the camera, so that they play with their bodies. The pictures stay on this computer."
```

On macOS 10.15 the game runs with keyboard and mouse only (ONNX Runtime 1.19 needs macOS 11); say so in the README.

---

## 6. Vision

### 6.1 The models

| Model | Role | Cost (i7-6600U, CPU) |
|---|---|---|
| **RTMO-t** body7 416x416 (MMPose, Apache-2.0) | **Detect** all bodies; runs only when a player slot is empty or its track was lost | ~70 ms |
| **RTMPose-t** body7 256x192 (MMPose, Apache-2.0) | **Track** each body top-down, in the box of its previous pose, every frame | ~13 ms per player |
| **YOLOX Body-Head-Hand** nano (PINTO model zoo, Apache-2.0) | **Hands** for the cursor (while a menu shows), heads | ~15–25 ms |
| YuNet (MIT), optional | Face landmarks in a head crop, to dress the face or to follow it cheaply (~1–3 ms) | |
| OSNet x0.25, optional | Appearance embedding of a person crop (clothes, not face) to re-identify a returning player | ~10 ms |

Next to each model: its `LICENSE` and a `<model>.json` with what you need to reproduce it:

```json
{
  "source": "https://download.openmmlab.com/mmpose/v1/projects/rtmo/onnx_sdk/rtmo-t_8xb32-600e_body7-416x416-f48f75cb_20231219.zip",
  "sha256": "…",
  "license": "Apache-2.0",
  "input": {"name": "input", "shape": [1, 3, 416, 416], "layout": "NCHW", "channels": "BGR", "range": "0..255", "pad": 114},
  "outputs": {"dets": "[1, N, 5] x0 y0 x1 y1 score", "keypoints": "[1, N, 17, 3] x y score"},
  "altered": "none",
  "notes": "Trained on Body7; check the datasets' terms before a commercial release."
}
```

**Check the training data of every model** before shipping and write the caveat into THIRD_PARTY_NOTICES (e.g. "MSMT17 is for research only.
Whether this restriction carries over to the trained weights is unclear.
Please take it into account before any commercial use.").
To make a fixed-size model dynamic (e.g. YuNet for head crops), rename the dimensions with `onnx` in a `dev/models/` script and record it as **Altered:** in the notices:

```python
model = onnx.load(source)
dims = model.graph.input[0].type.tensor_type.shape.dim
dims[2].ClearField("dim_value"); dims[2].dim_param = "H"
dims[3].ClearField("dim_value"); dims[3].dim_param = "W"
for output in model.graph.output:
    output.type.tensor_type.shape.dim[1].ClearField("dim_value")
    output.type.tensor_type.shape.dim[1].dim_param = f"anchors_{output.name.split('_')[-1]}"
del model.graph.value_info[:]
onnx.save(model, target)
```

### 6.2 Sessions

```csharp
public static class OnnxRuntimePrivacy
{
    private static int _done;
    /// Windows builds of ONNX Runtime emit telemetry; switch it off before any session.
    public static void DisableTelemetry()
    {
        if (Interlocked.Exchange(ref _done, 1) == 0) OrtEnv.Instance().DisableTelemetryEvents();
    }
}

OnnxRuntimePrivacy.DisableTelemetry();
using var options = new SessionOptions
{
    GraphOptimizationLevel = GraphOptimizationLevel.ORT_ENABLE_ALL,
    LogSeverityLevel = OrtLoggingLevel.ORT_LOGGING_LEVEL_ERROR,   // some models list initializers as inputs
    ExecutionMode = ExecutionMode.ORT_SEQUENTIAL,
    IntraOpNumThreads = 2,
    InterOpNumThreads = 1,
};
// Do not burn a core spinning while waiting for the next frame.
options.AddSessionConfigEntry("session.intra_op.allow_spinning", "0");
byte[] model = Godot.FileAccess.GetFileAsBytes("res://scenes/vision/rtmo/model/rtmo.onnx");   // in the bridge
if (IsLfsPointer(model)) throw new ModelsMissingException("run git lfs pull");
_session = new InferenceSession(model, options);
// Validate the input/output names and shapes here; throw a descriptive message; dispose on failure.

// Allocate the input once and reuse it every frame.
_input = new float[3 * InputSize * InputSize];
_inputValue = OrtValue.CreateTensorValueFromMemory(_input, new long[] { 1, 3, InputSize, InputSize });
_runOptions = new RunOptions();
```

* CPU only; it is enough, and it is the same everywhere.
* Add `*.onnx, *.json, *.po` to the `include_filter` of every export preset: Godot does not export files it does not know.
* **Degrade gracefully**: a missing optional model (faces) logs a warning; a missing required model sets the status `ModelsMissing`; the game stays playable with the keyboard.

### 6.3 RTMO: letterbox, BGR, decode

```csharp
public readonly struct Letterbox
{
    public readonly float Scale, PadX, PadY;
    public Letterbox(int width, int height, int size)
    {
        Scale = Math.Min((float)size / width, (float)size / height);
        PadX = (size - width * Scale) * 0.5f;
        PadY = (size - height * Scale) * 0.5f;
    }
    public float ToFrameX(float x) => (x - PadX) / Scale;
    public float ToFrameY(float y) => (y - PadY) / Scale;
}

/// Fit the frame into the planar BGR float input, centred, padded with 114, values 0..255.
public static Letterbox Preprocess(Frame frame, int size, float[] target)
{
    int plane = size * size;
    var box = new Letterbox(frame.Width, frame.Height, size);
    byte[] rgb = frame.Rgb; int width = frame.Width, height = frame.Height;
    float inverse = 1f / box.Scale;
    for (int y = 0; y < size; y++)
    {
        float sy = (y + 0.5f - box.PadY) * inverse - 0.5f;   // pixel centres, as cv2.resize
        bool insideY = sy >= -0.5f && sy <= height - 0.5f;
        sy = Math.Clamp(sy, 0f, height - 1);
        int y0 = (int)sy, y1 = Math.Min(y0 + 1, height - 1);
        float fy = sy - y0;
        int row0 = y0 * width * 3, row1 = y1 * width * 3;
        for (int x = 0; x < size; x++)
        {
            int index = y * size + x;
            float sx = (x + 0.5f - box.PadX) * inverse - 0.5f;
            if (!insideY || sx < -0.5f || sx > width - 0.5f)
            {
                target[index] = target[plane + index] = target[2 * plane + index] = PadValue;
                continue;
            }
            sx = Math.Clamp(sx, 0f, width - 1);
            int x0 = (int)sx, x1 = Math.Min(x0 + 1, width - 1);
            float fx = sx - x0;
            for (int c = 0; c < 3; c++)
            {
                float p0 = rgb[row0 + x0 * 3 + c] + (rgb[row0 + x1 * 3 + c] - rgb[row0 + x0 * 3 + c]) * fx;
                float p1 = rgb[row1 + x0 * 3 + c] + (rgb[row1 + x1 * 3 + c] - rgb[row1 + x0 * 3 + c]) * fx;
                // MMPose loads the images with OpenCV, so the model expects BGR: R goes to the last plane.
                target[(2 - c) * plane + index] = p0 + (p1 - p0) * fy;
            }
        }
    }
    return box;
}

public void Estimate(Frame frame, List<Pose> result)
{
    result.Clear();
    var box = Preprocess(frame, InputSize, _input);
    using var outputs = _session.Run(_runOptions, _inputNames, new[] { _inputValue }, _outputNames); // "dets", "keypoints"
    ReadOnlySpan<float> dets = outputs[0].GetTensorDataAsSpan<float>();
    ReadOnlySpan<float> keypoints = outputs[1].GetTensorDataAsSpan<float>();
    int persons = dets.Length / 5;
    for (int p = 0; p < persons; p++)
    {
        float score = dets[5 * p + 4];
        if (score < PersonThreshold) continue;   // 0.3
        var points = new Keypoint[Pose.KeypointCount];
        for (int k = 0; k < Pose.KeypointCount; k++)
        {
            int i = (p * Pose.KeypointCount + k) * 3;
            points[k] = new Keypoint(box.ToFrameX(keypoints[i]), box.ToFrameY(keypoints[i + 1]), keypoints[i + 2]);
        }
        var personBox = new BoundingBox(box.ToFrameX(dets[5 * p]), box.ToFrameY(dets[5 * p + 1]),
                                        box.ToFrameX(dets[5 * p + 2]), box.ToFrameY(dets[5 * p + 3]));
        result.Add(new Pose(points, personBox, score));
    }
}
```

YOLOX differs: the frame sits in the **top-left** corner, padded bottom and right; BGR 0..255; the "post" variant has NMS built in and outputs `[N, 7]` = (batch, class, score, x1, y1, x2, y2).

### 6.4 RTMPose: crop, normalize, SimCC, next box

```csharp
// Preprocess: crop the tracker box (nearest neighbour; black outside the frame), RGB, normalized.
static readonly float[] Mean = { 123.675f, 116.28f, 103.53f };
static readonly float[] Std = { 58.395f, 57.12f, 57.375f };
// target[c * plane + index] = (rgb[c] - Mean[c]) / Std[c];

// Decode: one row of bins per key point and axis, 2 bins per input pixel.
using var outputs = _session.Run(_runOptions, _inputNames, new[] { _inputValue }, _outputNames); // "simcc_x", "simcc_y"
var simccX = outputs[0].GetTensorDataAsSpan<float>();
var simccY = outputs[1].GetTensorDataAsSpan<float>();
int binsX = InputWidth * 2, binsY = InputHeight * 2;
for (int k = 0; k < Pose.KeypointCount; k++)
{
    var (bx, sx) = ArgMax(simccX.Slice(k * binsX, binsX));
    var (by, sy) = ArgMax(simccY.Slice(k * binsY, binsY));
    float x = _cx - _w / 2f + bx / 2f * _w / InputWidth;
    float y = _cy - _h / 2f + by / 2f * _h / InputHeight;
    points[k] = new Keypoint(x, y, Math.Min(sx, sy));
}

// The next box: from the confident key points, padded 1.25x, widened to 192:256.
public bool SetBox(ReadOnlySpan<Keypoint> points)
{
    // ...min/max over the points with score >= 0.4 into x0, y0, x1, y1, count...
    if (count < MinKeypoints /* 5 */) { _cx = float.NaN; return false; }   // lost → RTMO re-seeds
    float w = (x1 - x0) * 1.25f, h = (y1 - y0) * 1.25f;
    if (w * InputHeight > h * InputWidth) h = w * InputHeight / InputWidth; else w = h * InputWidth / InputHeight;
    _cx = (x0 + x1) / 2f; _cy = (y0 + y1) / 2f; _w = w; _h = h;
    return true;
}
```

Per frame: track every known player with RTMPose; run RTMO only for an empty or lost slot, and seed the tracker from its pose.
Thresholds: track 0.4, seed 0.3, RTMO person 0.3, signals 0.3.

### 6.5 Scheduling: fast every frame, slow when needed, adapt to the laptop

* The fast models run on every frame; the slow ones only when needed, at a rate that the fast path leaves room for:

  ```csharp
  /// Run the slow model as often as the fast path allows.
  public sealed class SlowModelScheduler
  {
      public double MinInterval = 0.05, MaxInterval = 0.25;
      public double FastBudget = 0.8;   // share of the frame time the fast path may use
      public double Step = 0.002, BackOff = 1.5, SlowShare = 0.5;
      private readonly object _lock = new();
      private double _interval = 0.1, _slowSeconds;

      public double Interval
      {
          get { lock (_lock) return Math.Clamp(Math.Max(_interval, _slowSeconds / SlowShare), MinInterval, MaxInterval); }
      }

      public void ObserveFast(double fastMs, double frameSeconds)
      {
          lock (_lock)
          {
              if (fastMs / 1000.0 > FastBudget * frameSeconds) _interval = Math.Min(MaxInterval, _interval * BackOff);
              else _interval = Math.Max(MinInterval, _interval - Step);
          }
      }

      public void ObserveSlow(double slowSeconds) { lock (_lock) _slowSeconds = slowSeconds; }
  }
  ```
* **Step the workload down on a weak laptop** by the P95 latency and the render fps (e.g. levels `(320x256, 15 Hz)`, `(320x256, 12 Hz)`, `(256x192, 10 Hz)`; detector budget 40 ms; render ≥ 50 fps; down after 2 s over budget, up after 8 s under half).
  Never below 256x192: YOLOX nano then misses about 10 % of the hands.
  Expose `OverBudget` so the game can tell the player.
* **A late result is matched in its own frame.**
  A slow model's result arrives after its frame: keep 1 s of the fast tracks' history, match the slow result against the fast detections of the *same* frame (±0.1 s), then follow forward to now.

### 6.6 Players

* **Pick the player** as the person with a head and the largest body unit, penalized by the distance from the middle: `score = unit * (1 - 0.5 * offCenter)`.
  *Why:* a parent standing next to the child must not steal the game.
* **Two players: split the frame.**
  Each owns half of the mirrored picture; a head crossing into the other half drops the track; the hands of the other half are ignored.
  Simple and robust.
* **Drop-in, up to four players: ByteTrack-like.**
  Constant-velocity prediction (horizon ≤ 0.3 s); gate `1.5 × max(track size, detection size)`; strong detections (≥ 0.5) match any track, weak ones (≥ 0.25) only confirmed tracks; 2 hits to confirm; lost after 0.5–0.8 s (a jump carries the head out of the frame).
  Use the appearance only where the motion is ambiguous, and embed only isolated players (no other head within 2.5 sizes).
* **A new head never gets a new identity while a player is away.**
  It takes over the identity of the lost player who looks most alike, or of the one lost most recently; no similarity threshold; remember for longer than a level.
  *Why:* children stepped out and back in to get a fresh avatar.
* Remember the state of a player who left (health, colour) until the next level.

### 6.7 Body units, home, hysteresis

* **Unit** = the widest shoulder width over the last 4 s (a monotonic deque); fall back to `ears × 2.2`, then `eyes × 4.4`, or use the head height.
  Every threshold is in units.
* **Keep reference positions in pixels, not in units.**
  *Why:* a home kept in units moved when the unit changed, and the boat spun while the player sat still.
* **Hysteresis everywhere**: a brake on at 0.3 units above the head and off at 0.0; a selection changes only 0.25 cards past its edge for 0.1 s.
* **Confirm a pose for 0.12 s** (0.3 s felt slow); keep a head for 0.5 s after a miss; keep the last value while a hand blinks out.
* **Settle and calm**: a gesture that presses a button counts only after 0.3 s of pointing and 1.5 s after the screen appeared.
  *Why:* the last jump of a level pressed the first card of the next screen.
* **Distance prompts**: head height ≤ 0.06 of the frame → "come closer"; ≥ 0.35 → "step back".
* A position held longer than the slowest move (e.g. 2 s down) is not a move: re-baseline.

### 6.8 Recording and replay

* `--record` writes JSONL of what the vision measured, **numbers only**; `--frames` adds JPGs through a bounded queue that drops when it falls behind.
* `--replay=<rec.jsonl>` feeds a recording instead of the camera.
* `dev/vision/record.gd` walks the player through a **CSV plan** of scripted steps (distances, called moves with beeps), so recordings are comparable; `dev/vision/analyze.py --extract` turns them into CSV fixtures for the unit tests.
* **Record only people who agreed; for children, their parents too.
  Never commit recordings.**

---

## 7. Smoothing: the 1€ filter

[Casiez et al., CHI 2012](https://gery.casiez.net/1euro/): little jitter when slow, little lag when fast.
Pass the time of the frame (4.3) and **measure the speed in body units**, so the filter behaves the same near and far from the camera.

`core/OneEuroFilter.cs` (2D, scale-invariant):

```csharp
public sealed class OneEuroFilter2
{
    public double MinCutoff = 1.0;
    public double Beta = 0.7;
    public double DerivativeCutoff = 1.0;

    private double _x = double.NaN, _y = double.NaN, _last = double.NaN, _speed;

    /// <param name="t">the time in seconds</param>
    /// <param name="unit">the size of the player in pixels (e.g., the shoulder width)</param>
    public (float X, float Y) Filter(double t, float x, float y, float unit)
    {
        if (double.IsNaN(_last) || double.IsNaN(_x)) { _x = x; _y = y; _last = t; return (x, y); }
        double dt = t - _last;
        if (dt <= 1e-6) return ((float)_x, (float)_y);
        _last = t;
        double scale = unit > 0f && !float.IsNaN(unit) ? unit : 100.0;
        double dx = x - _x, dy = y - _y;
        double speed = Math.Sqrt(dx * dx + dy * dy) / scale / dt;   // units per second
        _speed += Alpha(DerivativeCutoff, dt) * (speed - _speed);
        double alpha = Alpha(MinCutoff + Beta * _speed, dt);
        _x += alpha * dx;
        _y += alpha * dy;
        return ((float)_x, (float)_y);
    }

    public void Reset() { _x = _y = _last = double.NaN; _speed = 0; }

    private static double Alpha(double cutoff, double dt)
    {
        double tau = 1.0 / (2.0 * Math.PI * cutoff);
        return 1.0 / (1.0 + tau / dt);
    }
}
```

The same in GDScript, per axis, for UI code:

```gdscript
class OneEuro:
	extends RefCounted
	var min_cutoff := 1.2
	var beta := 4.0
	var derivative_cutoff := 1.0
	var _value := 0.0
	var _derivative := 0.0
	var _last := -1.0

	func filter(value: float, time: float) -> float:
		if _last < 0.0:
			_value = value
			_last = time
			return value
		var dt := time - _last
		if dt <= 0.000001:
			return _value
		_last = time
		var derivative := (value - _value) / dt
		_derivative += _alpha(derivative_cutoff, dt) * (derivative - _derivative)
		var cutoff := min_cutoff + beta * absf(_derivative)
		_value += _alpha(cutoff, dt) * (value - _value)
		return _value

	func _alpha(cutoff: float, dt: float) -> float:
		var tau := 1.0 / (TAU * cutoff)
		return 1.0 / (1.0 + tau / dt)
```

Start from these parameters and tune them on recordings:

| Signal | min_cutoff | beta | Speed in |
|---|---|---|---|
| Hand cursor | 1.0 | 0.7 | body units/s |
| Head position and size | 1.5 | 0.01 | pixels/s |
| Jump height (only remove jitter, keep the peaks) | 4.0 | 0.05 | pixels/s |
| Head roll | 1.0 | 0.5 | degrees/s |
| Pose key points (overlays) | 1.5 | 5.0 | pixels/s |

* Before filtering a cursor from a stale snapshot, extrapolate by the track's velocity, capped at 80 ms.
* Do **not** smooth the signals that detect motion (strokes, jumps) heavily; use dead zones, hysteresis and time constants there.

---

## 8. The hand cursor and the dwell buttons

### 8.1 Hands, not wrists

The cursor follows the **centre of the YOLOX hand box**; the pose's wrists only tell whose hand it is.
*Why:* the wrist lags, jitters with an open hand and is often wrong for children; the hand box is where the player thinks their hand is.

* Every raised hand gets its own glove cursor; hands are matched to cursors nearest-first; a jump of more than 0.25 (normalized) is a new hand; a cursor survives 0.3 s after its hand is lost.
* The **reach rectangle** (0.15, 0.1, 0.7, 0.6 of the mirrored picture; its bottom follows the shoulders) maps onto the whole screen.
  With two players, each has a half: `Rect2(0.04, 0.1, 0.42, 0.6)`, shifted by 0.5 for the second.
* **Sticky**: on a button the cursor moves only 35 % of the hand's offset from its centre, and leaves only 40–70 px past the edge (at most 35 % of the button's smaller side).
  Under a point, the smallest button wins.
* Do not ignore low hands: let the dwell time guard against resting hands.
  Keep buttons ≥ 200 px from the bottom and ≥ 60 px from the sides, where the hand detection is weak.
* Exactly one cursor per hand, drawn above every button; no shared static cursor state.
* The mouse and the keyboard always work too.

```gdscript
const STICKY_MARGIN := 70.0  # px
const STICKY_SHARE := 0.35
const _GLIDE_SECONDS := 0.08

static var reach := Rect2(0.15, 0.1, 0.7, 0.6)


func _to_screen(hand: Vector2) -> Vector2:  # hand: normalized, mirrored camera coordinates
	var local := (hand - reach.position) / reach.size
	return local.clamp(Vector2.ZERO, Vector2.ONE) * get_viewport_rect().size


func _update_cursor(cursor: Cursor, hand_on_screen: Vector2, delta: float) -> void:
	# hand_on_screen is already smoothed with the 1€ filter in the vision.
	var target := hand_on_screen
	var held := _button_under(hand_on_screen, cursor.held)
	if held != null:
		var middle := held.get_global_rect().get_center()
		target = middle + (hand_on_screen - middle) * STICKY_SHARE
	var glide := 1.0 - exp(-delta / _GLIDE_SECONDS)
	cursor.position += (target - cursor.position) * glide
	if held != cursor.held:
		if cursor.held != null:
			cursor.held.release()
		cursor.held = held
	if held != null:
		held.hold(delta)


## Sticky: stay on the held button until well past its edge; otherwise the smallest button wins.
func _button_under(at: Vector2, held: DwellButton) -> DwellButton:
	if held != null and held.is_visible_in_tree() and not held.disabled:
		var margin := minf(STICKY_MARGIN, minf(held.size.x, held.size.y) * STICKY_SHARE)
		if held.get_global_rect().grow(margin).has_point(at):
			return held
	var best: DwellButton = null
	for button: DwellButton in get_tree().get_nodes_in_group("dwell_buttons"):
		if not button.is_visible_in_tree() or button.disabled:
			continue
		var rect := button.get_global_rect()
		if rect.has_point(at) and (best == null or rect.get_area() < best.get_global_rect().get_area()):
			best = button
	return best
```

### 8.2 Dwell

* **2 s** to press (3 s for costly actions); the progress resets when the hand leaves; a ring fills clockwise from the top (the shader in section 9).
* **At most 4–6 big buttons per screen**, with pictures for children who cannot read yet.
* A destructive action needs a second, different confirmation (Delete → choose → Delete/Keep).
* A button speaks its text when a hand comes onto it, if read-aloud is on (section 20).
* If the hands are busy (on the floor, on a handle), choose by stepping left/right and press by a jump after 0.3 s of settling, or by an open→fist transition held 1.5 s; never by "looks like a fist" alone (hands on the floor look like fists).

```gdscript
class_name DwellButton
extends Button

const DWELL_SECONDS := 2.0
var dwell := 0.0  # 0..1


func _ready() -> void:
	add_to_group("dwell_buttons")


func hold(delta: float) -> void:  # by the cursor, every frame while held, with the game delta
	dwell += delta / DWELL_SECONDS
	if dwell >= 1.0:
		dwell = 0.0
		pressed.emit()


func release() -> void:
	dwell = 0.0
```

---

## 9. Button selection with shaders

A `ColorRect` 18 px larger than the button on every side, child of the button, `mouse_filter = IGNORE`, in the group `"glow"`, additive: a rounded-box SDF rim that pulses, a sheen sweeping across, and the dwell progress running around the rim.

`scenes/ui/selection_glow.gdshader`:

```glsl
shader_type canvas_item;
render_mode blend_add;

uniform vec2 button_size = vec2(320.0, 120.0);
uniform float margin = 18.0;
uniform float radius = 24.0;
uniform float hover : hint_range(0.0, 1.0) = 0.0;     // fades in and out
uniform float progress : hint_range(0.0, 1.0) = 0.0;  // the dwell
uniform vec4 glow : source_color = vec4(1.0, 0.78, 0.3, 1.0);
uniform vec4 ring : source_color = vec4(1.0, 0.95, 0.7, 1.0);

float rounded_box(vec2 p, vec2 half_size, float r) {
	vec2 q = abs(p) - half_size + r;
	return length(max(q, 0.0)) + min(max(q.x, q.y), 0.0) - r;
}

void fragment() {
	vec2 rect = button_size + 2.0 * margin;
	vec2 p = (UV - 0.5) * rect;  // pixels from the centre
	float d = rounded_box(p, button_size * 0.5, radius);

	float inside = 1.0 - smoothstep(-1.0, 1.0, d);
	float rim = exp(-abs(d) / 5.0);
	float pulse = 0.5 + 0.5 * sin(TIME * 4.0);

	// A diagonal sheen sweeping over the button.
	float along = p.x + p.y * 0.6;
	float span = button_size.x + button_size.y * 2.0;
	float band = fract(TIME * 0.5) * span * 1.5 - span * 0.5 - button_size.x * 0.25;
	float sheen = exp(-pow((along - band) / (button_size.y * 0.3), 2.0));

	// The dwell: the rim fills clockwise from the top.
	float angle = atan(p.x, -p.y);  // 0 at the top, growing clockwise
	float t = fract(angle / TAU + 1.0);
	float filled = step(t, progress) * step(0.001, progress);
	float ring_line = exp(-pow(d / 3.0, 2.0)) * filled;

	float a = hover * (inside * (0.05 + 0.22 * sheen) + rim * (0.35 + 0.3 * pulse));
	vec3 color = glow.rgb * a + ring.rgb * ring_line;
	COLOR = vec4(color, clamp(max(a, ring_line), 0.0, 1.0));
}
```

`scenes/ui/selection_glow.gd`:

```gdscript
extends ColorRect

const MARGIN := 18.0
var _hover := 0.0
@onready var _button: DwellButton = get_parent()


func _ready() -> void:
	mouse_filter = Control.MOUSE_FILTER_IGNORE
	add_to_group("glow")
	material = material.duplicate()  # one set of uniforms per button


func _process(delta: float) -> void:
	position = Vector2(-MARGIN, -MARGIN)
	size = _button.size + Vector2(2 * MARGIN, 2 * MARGIN)
	var target := 1.0 if (_button.is_hovered() or _button.dwell > 0.0) else 0.0
	_hover = lerpf(_hover, target, 1.0 - exp(-delta * 12.0))  # independent of the frame rate
	var shader := material as ShaderMaterial
	shader.set_shader_parameter("button_size", _button.size)
	shader.set_shader_parameter("hover", _hover)
	shader.set_shader_parameter("progress", _button.dwell)
```

The hovered button may grow by a few pixels; its contents do not move.
Keep buttons at least `MARGIN` from the screen edge (the overflow contract allows the glow its margin and nothing more).
For the dim backdrop behind a menu, cut feathered holes with the same SDF (up to 32 rounded rectangles, `smoothstep`, a residue of 0.08) where the world must show through.

---

## 10. The player is always visible

**The players see themselves, and see that the game sees them.**

* **The dim, mirrored camera picture fills the screen behind everything**: alpha about 0.2 during play, 0.55 behind menus (so the hand is under its cursor).
  Show the `CameraTexture` directly with `flip_h` and `stretch_mode = KEEP_ASPECT_COVERED`; the GPU mirrors, not the CPU.
* **Avatars and markers are drawn exactly on the players**: a ship on the head, a helmet that tilts with the head, a glove on each hand, a halo on the head.
  Map detections with the **same cover crop** as the picture:
  ```csharp
  /// Map a normalized, mirrored camera position onto the screen, as the cover-cropped picture shows it.
  public static Vector2 ToScreen(Vector2 position, Vector2 frameSize, Rect2 rect)
  {
      Vector2 pixel = position * frameSize;
      float scale = Math.Max(rect.Size.X / frameSize.X, rect.Size.Y / frameSize.Y);
      Vector2 visibleSize = rect.Size / scale;
      Vector2 crop = (frameSize - visibleSize) * 0.5f;
      return (rect.Position + (pixel - crop) * scale).Clamp(rect.Position, rect.End);
  }
  ```
  *Why:* a full-screen mirror with the avatar elsewhere on the screen confused players; with the avatar on the body it works.
* Where the game world is not the camera picture (a map, a boat), show a **camera panel** in an upper corner (512x384, mirrored, with the skeleton and a big command word: "STEP IN FRONT", "HOLD STILL", "READY"), toggled with F1, and keep the lower corners free.
* Show the player's own head, not an avatar's face.

### 10.1 No GUI/HUD element ever covers a player

* **Hide counters from children**: the HUD keeps its counts (the tests read them) but shows only what a child acts on.
* **Lay out the HUD around the players**, every frame (`scenes/ui/hud_layout.gd`):
  * pieces in **priority order**: a piece never moves onto pieces placed before it;
  * candidate spots: home, home mirrored, a grid along the edges in eighths, and the spots right beside each player; sorted by distance from home;
  * **hysteresis**: a piece leaves when a player comes within `NEAR`, and goes to a spot (also back home) only if every player is `CLEAR` away;
  * it **glides** there over 1.2 s (smoothstep); a newly shown piece appears in place;
  * if nothing is free: the spot that covers the players least, then the earlier pieces least.

```gdscript
class_name HudLayout
extends Node

const NEAR := 8.0  # px: leave when a player is this close
const CLEAR := 32.0  # px: go only where every player is this far
const GLIDE_SECONDS := 1.2


class Piece:
	extends RefCounted
	var control: Control
	var home: Vector2
	var target: Vector2
	var start: Vector2
	var t := 1.0
	var was_visible := false

	func _init(a_control: Control) -> void:
		control = a_control
		home = a_control.position
		target = home
		start = home


var pieces: Array[Piece] = []  # in priority order


func add(control: Control) -> void:
	control.add_to_group("hud")
	pieces.append(Piece.new(control))


func layout(players: Array[Rect2], delta: float) -> void:
	var screen := get_viewport().get_visible_rect()
	var placed: Array[Rect2] = []
	for piece in pieces:
		var shown := piece.control.is_visible_in_tree()
		if not shown:
			piece.was_visible = false
			continue
		var size := piece.control.size
		var current := Rect2(piece.target, size)
		var home := Rect2(piece.home, size)
		if _free(home, players, placed, CLEAR):
			_go(piece, piece.home)
		elif not _free(current, players, placed, NEAR):
			_go(piece, _best_spot(piece, players, placed, screen))
		if not piece.was_visible:  # newly shown: appear in place, no glide
			piece.t = 1.0
			piece.start = piece.target
		piece.was_visible = true
		piece.t = minf(1.0, piece.t + delta / GLIDE_SECONDS)
		piece.control.position = piece.start.lerp(piece.target, smoothstep(0.0, 1.0, piece.t))
		placed.append(Rect2(piece.target, size))


func _go(piece: Piece, spot: Vector2) -> void:
	if spot.is_equal_approx(piece.target):
		return
	piece.start = piece.control.position
	piece.target = spot
	piece.t = 0.0


func _free(rect: Rect2, players: Array[Rect2], placed: Array[Rect2], margin: float) -> bool:
	for player in players:
		if rect.grow(margin).intersects(player):
			return false
	for other in placed:
		if rect.intersects(other):
			return false
	return true


func _best_spot(piece: Piece, players: Array[Rect2], placed: Array[Rect2], screen: Rect2) -> Vector2:
	var size := piece.control.size
	var spots := _candidates(piece, players, screen)
	spots.sort_custom(func(a: Vector2, b: Vector2) -> bool:
		return a.distance_squared_to(piece.home) < b.distance_squared_to(piece.home))
	for spot in spots:
		if _free(Rect2(spot, size), players, placed, CLEAR):
			return spot
	var best := piece.target
	var least := INF
	for spot in spots:  # nothing free: cover the players least, then the earlier pieces least
		var rect := Rect2(spot, size)
		var cover := 0.0
		for player in players:
			cover += 1000.0 * rect.intersection(player).get_area()
		for other in placed:
			cover += rect.intersection(other).get_area()
		if cover < least:
			least = cover
			best = spot
	return best


func _candidates(piece: Piece, players: Array[Rect2], screen: Rect2) -> Array[Vector2]:
	const PAD := 24.0
	var size := piece.control.size
	var room := screen.size - size - Vector2(PAD, PAD)
	var spots: Array[Vector2] = [piece.home, Vector2(screen.size.x - size.x - piece.home.x, piece.home.y)]
	for i in range(9):
		var f := i / 8.0
		spots.append(Vector2(PAD + f * (room.x - PAD), PAD))
		spots.append(Vector2(PAD + f * (room.x - PAD), room.y))
		spots.append(Vector2(PAD, PAD + f * (room.y - PAD)))
		spots.append(Vector2(room.x, PAD + f * (room.y - PAD)))
	for player in players:
		spots.append(Vector2(player.position.x - size.x - CLEAR, player.position.y))
		spots.append(Vector2(player.end.x + CLEAR, player.position.y))
	var inside: Array[Vector2] = []
	for spot in spots:
		inside.append(spot.clamp(Vector2(PAD, PAD), room))
	return inside
```

* **Test it two ways**: the end-to-end contract (12.5) checks every frame, with a grace of `GLIDE_SECONDS + 0.8` s for the glide; a **HUD fuzz test** puts the players at random places (edges, beside every piece, 1 and 2 players, every HUD state) with a fixed seed and checks that after the glide nothing covers them.

---

## 11. Translations with PO and POT

Translate once the game works, before the first release (1); while prototyping, only write every text as a whole sentence in `tr()`.

### 11.1 In the code

* Every text a player sees goes through `tr("...")`: **a whole sentence with named placeholders**, never glued from pieces, and the literal in one piece (the extraction reads it from the code).
* `# TRANSLATORS:` right above the call reaches the translators; **"at most N characters"** in it is enforced by `check.py`.
* Placeholders are filled by `Grammar.format` (11.4), not by `%` or `+`.

```gdscript
# TRANSLATORS: a button, at most 20 characters
start_button.text = tr("Start the journey")
# TRANSLATORS: {port} is the name of a port; {count} a number
label.text = Grammar.format(tr("Sail to {port@to} with {count} {count|barrel|barrels}."), {"port": port.name, "count": 3})
```

### 11.2 The files

* `translations/languages.json` is the **single registry**: one entry per language with its `code`, its `name` in itself, a `prompt` ("Choose your language" in that language), the `alphabet` for the on-screen name keyboard, and the `voice` locale for read-aloud.
  Adding a language is one entry plus running `extract.py`, then setting the `Plural-Forms` header.
* `translations/messages.pot` is generated; `translations/<code>.po` are merged from it, keeping the translations; unused ones stay as obsolete for when a message comes back.
* A new translation (also a machine draft) is marked **fuzzy** until a native speaker reviewed it in the game; then the flag is removed.
  `check.py` reports the coverage per language.

### 11.3 Loading at runtime

**Godot's PO importer skips fuzzy entries** (they would show in English), so the game loads the `.po` files itself, strips the fuzzy flags, and registers the translations from `languages.json`.
Ship the raw `.po`: add `*.po` to the `include_filter` of every export preset, and do **not** list them in `project.godot`.

`scenes/languages/languages.gd` (autoload `Languages`):

```gdscript
extends Node

const _REGISTRY := "res://translations/languages.json"
const _TRANSLATIONS_DIR := "user://translations"

var languages: Array[Language] = []
var _added: Array[Translation] = []


class Language:
	extends RefCounted
	var code: String
	var name: String
	var prompt: String
	var alphabet: String
	var voice: String


func _enter_tree() -> void:
	for entry: Dictionary in JSON.parse_string(FileAccess.get_file_as_string(_REGISTRY)):
		var language := Language.new()
		language.code = entry["code"]
		language.name = entry["name"]
		language.prompt = entry.get("prompt", "")
		language.alphabet = entry.get("alphabet", "")
		language.voice = entry.get("voice", "")
		languages.append(language)
		if language.code != "en":
			_add(language.code)


## Load a .po without its fuzzy flags: Godot skips fuzzy messages, but we show drafts until reviewed.
func _add(code: String) -> void:
	var kept: PackedStringArray = []
	for line in FileAccess.get_file_as_string("res://translations/%s.po" % code).split("\n"):
		if line.begins_with("#,") and "fuzzy" in line:
			var flags := line.substr(2).replace("fuzzy", "").replace(",", " ").strip_edges()
			if flags != "":
				kept.append("#, " + ", ".join(flags.split(" ", false)))
			continue
		kept.append(line)
	DirAccess.make_dir_recursive_absolute(_TRANSLATIONS_DIR)
	# Every running game writes a file of its own (several may start at once, e.g., in the tests).
	var target := "%s/%s_%d.po" % [_TRANSLATIONS_DIR, code, OS.get_process_id()]
	var file := FileAccess.open(target, FileAccess.WRITE)
	file.store_string("\n".join(kept))
	file.close()
	var translation: Translation = ResourceLoader.load(target, "", ResourceLoader.CACHE_MODE_IGNORE)
	DirAccess.remove_absolute(target)
	var addressed := AddressedTranslation.new()
	addressed.inner = translation
	addressed.locale = translation.locale
	TranslationServer.add_translation(addressed)
	_added.append(addressed)


func _exit_tree() -> void:
	# Remove the script translations, otherwise the engine crashes when it frees them.
	for translation in _added:
		TranslationServer.remove_translation(translation)
	_added.clear()
```

### 11.4 Plural, case and gender

Write the grammar **inside the translation**, so English stays one message and only the languages that need forms write them:

* `{count|seat|seats}` picks the noun form for the number of `{count}` by the plural rule of the language (one/other; Slavic one/few/many; French counts 0 as one).
* `{port@to}` uses the translation of the name `port` with the context `"to"`, which includes the preposition ("в Пулу", "na Hvar"); empty falls back to the plain name.
  The extraction adds one message per name × context, with an example for the translator.
* `{crew|captain|captain (f)|captains|captains (f)}` follows the gender and number of the addressed players; an `AddressedTranslation` resolves it when the message is translated, so it works in every text.
* Spoken lines avoid forms that reveal the speaker's gender.

`scenes/languages/grammar.gd`:

```gdscript
class_name Grammar
extends RefCounted

## How the players are addressed: 0 male, 1 female, 2 several, 3 several female.
static var crew := 0
## For the layout tests: always take the longest form.
static var longest_crew_forms := false

static var _placeholder := RegEx.create_from_string("\\{([a-z_]+)(@[a-z_]+)?(\\|[^{}]*)?\\}")
static var _crew := RegEx.create_from_string("\\{crew\\|([^{}]*)\\}")


## Fill the placeholders of a translated sentence.
static func format(text: String, args: Dictionary) -> String:
	var result := ""
	var at := 0
	for found in _placeholder.search_all(text):
		result += text.substr(at, found.get_start() - at)
		at = found.get_end()
		var key := found.get_string(1)
		if not args.has(key):
			result += found.get_string()
			continue
		var value: Variant = args[key]
		var context := found.get_string(2).substr(1)
		var forms := found.get_string(3)
		if forms != "":
			var options := forms.substr(1).split("|")
			result += options[mini(plural_index(int(value)), options.size() - 1)]
		elif value is String or value is StringName:
			result += _name(String(value), context)
		else:
			result += str(value)
	return result + text.substr(at)


static func _name(name: String, context: String) -> String:
	if context != "":
		var form := String(TranslationServer.translate(name, context))
		if form != "" and form != name:
			return form
	return String(TranslationServer.translate(name))


## Replace every {crew|...} with the form for the addressed players.
static func address(text: String) -> String:
	var result := ""
	var at := 0
	for found in _crew.search_all(text):
		result += text.substr(at, found.get_start() - at)
		at = found.get_end()
		var forms := found.get_string(1).split("|")
		if longest_crew_forms:
			var longest := ""
			for form in forms:
				longest = form if form.length() > longest.length() else longest
			result += longest
		else:
			result += forms[mini(crew, forms.size() - 1)]
	return result + text.substr(at)


static func plural_index(count: int) -> int:
	var language := TranslationServer.get_locale().left(2)
	var n := absi(count)
	if language == "fr":
		return 0 if n <= 1 else 1
	if language in ["ru", "hr", "sr", "bs", "uk"]:
		var last := n % 10
		var last_two := n % 100
		if last == 1 and last_two != 11:
			return 0
		if last >= 2 and last <= 4 and (last_two < 12 or last_two > 14):
			return 1
		return 2
	return 0 if n == 1 else 1
```

`scenes/languages/addressed_translation.gd`:

```gdscript
class_name AddressedTranslation
extends Translation

var inner: Translation


func _get_message(src_message: StringName, context: StringName) -> StringName:
	var text := String(inner.get_message(src_message, context))
	return StringName(Grammar.address(text)) if text != "" else &""
```

### 11.5 Extraction and checks

`dev/translations/extract.py` (needs `polib`) collects the `tr("...")` and `tr("...", "context")` calls with their `# TRANSLATORS:` comments, the text fields of `data/*.json`, and one message per name × context for `{x@context}`; it writes the POT and merges every PO.
With `--check` it writes nothing and fails if the files are out of date.

```python
#!/usr/bin/env python3
"""
Collect the translatable messages into translations/messages.pot and merge them into
translations/<code>.po. With --check, write nothing and fail if the files are out of date.

Usage: venv/bin/python dev/translations/extract.py [--check]
"""
import argparse
import json
import pathlib
import re
import sys
from typing import Dict, List, Optional, Tuple

import polib

REPO_ROOT = pathlib.Path(__file__).resolve().parent.parent.parent
TRANSLATIONS_DIR = REPO_ROOT / "translations"
LANGUAGES = [
    entry["code"]
    for entry in json.loads((TRANSLATIONS_DIR / "languages.json").read_text(encoding="utf-8"))
    if entry["code"] != "en"
]

_STRING = r'"((?:[^"\\]|\\.)*)"'
TR_RE = re.compile(r"\btr\(\s*" + _STRING + r"(?:\s*,\s*" + _STRING + r")?\s*\)", re.DOTALL)
TRANSLATORS_RE = re.compile(r"#\s*TRANSLATORS:\s*(.*)$")

#: The translatable fields of data/*.json, and what they are (for the translators).
DATA_FIELDS = {
    "name": "a name",
    "description": "what it does, one short sentence",
    "title": "a title, a few words",
}

#: The names which are inflected: (data file, field) -> contexts, with an example for the translators.
FORMS = {("data/ports.json", "name"): ["to", "in"]}
FORM_EXAMPLES = {"to": "to the port, e.g. 'в Пулу', 'na Hvar'", "in": "in the port, e.g. 'в Пуле', 'u Puli'"}

Key = Tuple[str, str]  # (context, message id)


def _unescape(text: str) -> str:
    return text.encode("utf-8").decode("unicode_escape").encode("latin-1").decode("utf-8")


def _add(entries: Dict[Key, polib.POEntry], msgid: str, context: str, relative: str,
         comment: Optional[str]) -> None:
    entry = entries.get((context, msgid))
    if entry is None:
        entry = polib.POEntry(msgid=msgid, msgstr="")
        if context:
            entry.msgctxt = context
        entries[(context, msgid)] = entry
    # Only the file, not the line, so that the translation files do not change whenever code moves.
    if (relative, "") not in entry.occurrences:
        entry.occurrences.append((relative, ""))
    if comment and comment not in (entry.comment or ""):
        entry.comment = (entry.comment + "\n" if entry.comment else "") + comment


def _collect_scripts(entries: Dict[Key, polib.POEntry]) -> None:
    for path in sorted((REPO_ROOT / "scenes").rglob("*.gd")):
        text = path.read_text(encoding="utf-8")
        relative = path.relative_to(REPO_ROOT).as_posix()
        lines = text.splitlines()
        for match in TR_RE.finditer(text):
            msgid = _unescape(match.group(1))
            if not msgid:
                continue
            context = _unescape(match.group(2)) if match.group(2) is not None else ""
            line = text.count("\n", 0, match.start()) + 1
            comment = None
            for previous in reversed(lines[: line - 1]):
                found = TRANSLATORS_RE.search(previous.strip())
                if found:
                    comment = found.group(1)
                    break
                if previous.strip() and not previous.strip().startswith("#"):
                    break
            _add(entries, msgid, context, relative, comment)


def _collect_data(entries: Dict[Key, polib.POEntry], value: object, relative: str, field: str = "") -> None:
    if isinstance(value, dict):
        for key, item in value.items():
            _collect_data(entries, item, relative, key)
    elif isinstance(value, list):
        for item in value:
            _collect_data(entries, item, relative, field)
    elif isinstance(value, str) and field in DATA_FIELDS and value:
        _add(entries, value, "", relative, DATA_FIELDS[field])
        for context in FORMS.get((relative, field), []):
            _add(entries, value, context, relative, f"the name {FORM_EXAMPLES[context]}; empty for the plain name")


def make_template() -> polib.POFile:
    entries: Dict[Key, polib.POEntry] = {}
    _collect_scripts(entries)
    for path in sorted((REPO_ROOT / "data").rglob("*.json")):
        relative = path.relative_to(REPO_ROOT).as_posix()
        _collect_data(entries, json.loads(path.read_text(encoding="utf-8")), relative)
    template = polib.POFile(wrapwidth=100)
    template.metadata = {"Project-Id-Version": REPO_ROOT.name, "MIME-Version": "1.0",
                         "Content-Type": "text/plain; charset=UTF-8", "Content-Transfer-Encoding": "8bit"}
    for entry in entries.values():
        template.append(entry)
    return template


def merge(template: polib.POFile, existing: Optional[polib.POFile], language: str) -> polib.POFile:
    old = {(e.msgctxt or "", e.msgid): e for e in existing} if existing is not None else {}
    result = polib.POFile(wrapwidth=100)
    result.metadata = dict(template.metadata)
    result.metadata["Language"] = language
    if existing is not None:
        for name in ("Last-Translator", "Language-Team", "Plural-Forms"):
            if name in existing.metadata:
                result.metadata[name] = existing.metadata[name]
    used = set()
    for entry in template:
        merged = polib.POEntry(msgid=entry.msgid, msgctxt=entry.msgctxt, msgstr="",
                               occurrences=list(entry.occurrences), comment=entry.comment)
        previous = old.get((entry.msgctxt or "", entry.msgid))
        if previous is not None:
            used.add((entry.msgctxt or "", entry.msgid))
            merged.msgstr = previous.msgstr
            merged.flags = [flag for flag in previous.flags if flag != "obsolete"]
            merged.tcomment = previous.tcomment
        result.append(merged)
    # Keep unused translations as obsolete, so a message which comes back needs no new translation.
    for key, previous in old.items():
        if key not in used and previous.msgstr:
            previous.obsolete = True
            result.append(previous)
    return result


def main() -> int:
    parser = argparse.ArgumentParser(description=__doc__, formatter_class=argparse.RawDescriptionHelpFormatter)
    parser.add_argument("--check", action="store_true", help="Fail if the translation files are out of date.")
    args = parser.parse_args()
    template = make_template()
    outputs: List[Tuple[pathlib.Path, str]] = [(TRANSLATIONS_DIR / "messages.pot", str(template))]
    for language in LANGUAGES:
        path = TRANSLATIONS_DIR / f"{language}.po"
        existing = polib.pofile(str(path), wrapwidth=100) if path.exists() else None
        outputs.append((path, str(merge(template, existing, language))))
    stale = [path for path, text in outputs if not path.exists() or path.read_text("utf-8") != text]
    if args.check:
        for path in stale:
            print(f"{path.relative_to(REPO_ROOT)} is out of date; run dev/translations/extract.py", file=sys.stderr)
        return 1 if stale else 0
    for path, text in outputs:
        if path in stale:
            path.write_text(text, encoding="utf-8")
            print(f"Wrote {path.relative_to(REPO_ROOT)}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

`dev/translations/check.py` fails on what would break the game, and prints the coverage (`--verbose` lists the open messages):

```python
PLACEHOLDER_RE = re.compile(r"\{([a-z_]+)(?:@[a-z_]+)?\}")
AGREEMENT_RE = re.compile(r"\{([a-z_]+)\|[^{}]*\}")
CREW_RE = re.compile(r"\{crew\|([^{}]*)\}")
LIMIT_RE = re.compile(r"at most (\d+) characters")

for entry in polib.pofile(str(path)):
    if entry.obsolete or not entry.msgstr:
        continue
    where = entry.occurrences[0][0] if entry.occurrences else "?"
    expected = sorted(set(PLACEHOLDER_RE.findall(entry.msgid)))
    found = sorted(set(PLACEHOLDER_RE.findall(entry.msgstr)))
    if expected != found:
        errors.append(f"{code}: {where}: the placeholders {found} do not match {expected} of {entry.msgid!r}")
    for agreed in AGREEMENT_RE.findall(entry.msgstr):
        if agreed != "crew" and agreed not in expected:
            errors.append(f"{code}: {where}: the word agrees with {{{agreed}}}, which {entry.msgid!r} does not have")
    for forms in CREW_RE.findall(entry.msgstr):
        if not 2 <= len(forms.split("|")) <= 4:
            errors.append(f"{code}: {where}: {{crew|{forms}}} needs 2 to 4 forms")
    limit = LIMIT_RE.search(entry.comment or "")
    shown = CREW_RE.sub(lambda m: max(m.group(1).split("|"), key=len), entry.msgstr)
    if limit and len(shown) > int(limit.group(1)):
        errors.append(f"{code}: {where}: {len(shown)} characters, but at most {limit.group(1)} fit: {shown!r}")
```

Both run in `precommit.py`.
The character limits are a first line of defence; the real checks are the tests in 12.5: the layout of every screen **in every language** with the longest forms and the longest names a player can type, the font covering every letter of every `.po`, and every shown text being in the language of the game.

---

## 12. Testing

Write the tests once the game works (1); while prototyping, `precommit.py --prototype` checks only the formatting, lint and types.

| Layer | How | Runs |
|---|---|---|
| Vision and rules core | xUnit + **CsCheck** on the linked Godot-free sources | `dotnet test` |
| Real models | real ONNX files on public-domain photos padded to 640x480 | `dotnet test` |
| Recorded motion | CSV fixtures from consented recordings | `dotnet test` |
| One mechanic | a test scene per mechanic with scripted players; also an editor tool | headless Godot |
| Layout | every screen × every language × longest input; HUD fuzz | headless Godot |
| **The whole game** | **seeded end-to-end bot through a fake vision, contracts every frame, restart from disk, all languages** | headless Godot |
| Old saves | frozen saves of released versions, loaded and played on | headless Godot |
| Every script parses / every scene starts | `--check-only`; `--quit-after 120` | precommit |
| Before a release | soak tests (minutes per level: stuck, growing object counts); render tests with a window | by hand |

### 12.1 Unit tests of the Godot-free core

The game assembly cannot load outside the engine: **link the sources** into a plain test project.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <IsPackable>false</IsPackable>
  </PropertyGroup>
  <ItemGroup>
    <Compile Include="../../scenes/vision/core/*.cs" LinkBase="Linked/vision" />
    <None Include="../../scenes/vision/**/model/*.onnx" LinkBase="models" CopyToOutputDirectory="PreserveNewest" />
    <None Include="data/**" CopyToOutputDirectory="PreserveNewest" />
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.*" />
    <PackageReference Include="xunit" Version="2.9.3" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.*" />
    <PackageReference Include="CsCheck" Version="4.*" />
    <PackageReference Include="StbImageSharp" Version="2.*" />
    <PackageReference Include="Microsoft.ML.OnnxRuntime" Version="1.19.2" />   <!-- the game's version -->
  </ItemGroup>
</Project>
```

* Name tests as sentences: `Sitting_upright_after_a_long_lean_does_not_steer_the_other_way`.
* Feed synthetic motion:
  ```csharp
  private static Func<double, float> Jump(float height, double duration = 0.5)
      => s => s >= duration ? Standing : Standing - height * Size * (float)Math.Sin(Math.PI * s / duration);

  [Fact]
  public void Jumps_in_a_row_count_each()
  {
      var gauge = new JumpGauge();
      double t = 0;
      int jumps = 0;
      for (int i = 0; i < 10; i++) jumps += Feed(gauge, ref t, 0.8, Jump(1.0f));
      Assert.Equal(10, jumps);
  }
  ```
* Test the real models on **public-domain photos** (NASA) padded into black 640x480 frames: IoU against boxes measured by eye, heads seen from behind, an empty frame gives nothing, and embeddings separate people.
* When Python and C# compute the same thing, test **parity on a fixture generated by Python**; never keep a second copy of the logic in Python.

### 12.2 Property-based tests and fuzzing (CsCheck)

Further reading: [property-based testing](https://en.wikipedia.org/wiki/Software_testing#Property_testing) (and its origin, [QuickCheck](https://en.wikipedia.org/wiki/QuickCheck)), [fuzzing](https://en.wikipedia.org/wiki/Fuzzing).

A property says what must hold **for every input** ("the output stays within the range of the inputs"), instead of one example.
Generate the inputs; CsCheck shrinks a failure to a minimal case and prints the seed to replay it.

```csharp
using CsCheck;

public class OneEuroFilterProperties
{
    [Fact]
    public void The_output_stays_within_the_range_of_the_inputs() =>
        Gen.Select(Gen.Float[-1000f, 1000f].Array[1, 200], Gen.Double[0.001, 0.2])
            .Sample((values, dt) =>
            {
                var filter = new OneEuroFilter2();
                float min = values.Min(), max = values.Max();
                double t = 0;
                foreach (float v in values)
                {
                    var (x, _) = filter.Filter(t, v, -v, unit: 100f);
                    t += dt;
                    if (x < min - 1e-3 || x > max + 1e-3) return false;
                }
                return true;
            });

    [Fact]
    public void A_constant_input_is_a_fixed_point() =>
        Gen.Select(Gen.Float[-1000f, 1000f], Gen.Int[1, 500])
            .Sample((value, n) =>
            {
                var filter = new OneEuroFilter2();
                (float X, float Y) last = default;
                for (int i = 0; i < n; i++) last = filter.Filter(i / 30.0, value, value, 100f);
                return Math.Abs(last.X - value) < 1e-3;
            });
}

public class MeterProperties
{
    /// Body units: the same motion, near or far from the camera, counts the same.
    [Fact]
    public void Strokes_do_not_depend_on_the_distance_to_the_camera() =>
        Gen.Select(Gen.Float[0.4f, 2.5f], Gen.Int[1, 12], Gen.UInt)
            .Sample((scale, strokes, seed) =>
                Rowing.Count(Rowing.Synthesize(strokes, seed, scale: 1f))
                == Rowing.Count(Rowing.Synthesize(strokes, seed, scale: scale)));

    /// Fuzz: garbage key points (NaN, out of the frame, teleports, dropouts) never throw
    /// and never give a steering outside [-1, 1].
    [Fact]
    public void Garbage_never_breaks_the_meter() =>
        Gen.Select(Gen.Float, Gen.Float, Gen.Float[0f, 1f], Gen.Bool).Array[1, 600]
            .Sample(samples =>
            {
                var meter = new RowingMeter();
                double t = 0;
                foreach (var (x, y, score, seen) in samples)
                {
                    t += 1.0 / 30;
                    meter.Update(t, seen ? Pose.Synthetic(x, y, score) : null);
                    if (float.IsNaN(meter.Steer) || meter.Steer < -1f || meter.Steer > 1f) return false;
                }
                return true;
            });
}
```

Typical properties: invariance to scale and position (body units), bounds of every output, no exception on any input, idempotence of save → load → save, and "cooperation is never slower than playing alone" for the rules.

### 12.3 Contracts in the code

Further reading: [design by contract](https://en.wikipedia.org/wiki/Design_by_contract): preconditions (what a function expects), postconditions (what it promises) and invariants (what always holds), checked by the code itself.

* Check the preconditions at the boundaries and fail with a message that says what was expected and what came (as `YcbcrConverter` does).
* In GDScript, `assert(condition, message)` runs in debug builds and the tests and is stripped from release exports: use it for the invariants of the rule classes:
  ```gdscript
  func step(dt: float) -> void:
  	assert(dt >= 0.0, "Expected a non-negative dt, got %f" % dt)
  	...
  	assert(silver >= 0, "Silver went negative: %d" % silver)
  	assert(cargo.size() <= ship.hold, "Cargo %d over the hold %d" % [cargo.size(), ship.hold])
  ```
* The GUI, language and visibility contracts depend on the layout, the language and the players; the end-to-end test checks them on every frame (12.5).

### 12.4 The end-to-end bot

The bot is a [fuzzer](https://en.wikipedia.org/wiki/Fuzzing) of the whole game, and the contracts of 12.5 are its oracle: random but plausible inputs, checked against [invariants](https://en.wikipedia.org/wiki/Design_by_contract) on every frame.

A seeded bot plays **the whole game** through a fake of the vision bridge, at high speed, and checks the contracts on every frame and after every substep.
It is the most valuable test we have: its first runs found crashes, a screen replaced on the frame it appeared, cards wider than the screen, text overflowing its cards, a cursor on the wall clock, invisible walls from single-precision polygon tests, and physics push-outs that broke the speed limit.

What it does:

* clicks only through the fake hands (`point_at`) and the dwell, and asserts that **each button got exactly one press**;
* plays every level of every chapter, with plausible random choices, some deliberate mistakes (about 12 %) and some quitting mid-way (about 6 %);
* **switches the language every few levels** (shuffled by the seed), so one run covers every language;
* **restarts the game** halfway and checks that the language and the journey come back from disk exactly;
* runs with `--fixed-fps 60`, `Engine.time_scale = 10` and `Engine.max_fps = 0`; where a level needs more speed, it calls the level's `_process` for extra substeps and checks the contracts after each; it seeds the global RNG too;
* detects being stuck: a card held but not chosen, no progress towards the goal, a task not done within N legs, a wall-clock limit;
* **fails on any engine error**, even one the game survives;
* **always logs** everything (the seed, every screen, every choice, every broken contract, every engine error) to `user://tests/logs/<test>_<seed>.log`, and **deletes the log itself when the run passed**; on a failure it keeps the log and prints its path.
  *Why:* a long fuzz run never has to be repeated just to see what went wrong.

`tests/godot/end2end/fake_vision.gd` (the same method names as `VisionService.cs`):

```gdscript
# gdlint: disable=function-name
extends Node

var hand_tracking := false
var hand := Vector2.INF  # normalized, mirrored camera coordinates
var strokes: Array[int] = [0, 0]
var steer: Array[float] = [0.0, 0.0]
var brake := false
var players := 1
var player_rects: Array[Rect2] = [Rect2(Vector2(760, 300), Vector2(400, 700))]


func GetStatus() -> int:
	return 0  # Ready


func HasPlayer(p: int) -> bool:
	return p < players


func GetStrokes(p: int) -> int:
	return strokes[p]


func GetSteer(p: int) -> float:
	return steer[p]


func GetBrake(_p: int) -> bool:
	return brake


func GetHands() -> Array[Vector2]:
	if hand_tracking and hand != Vector2.INF:
		return [hand]
	return []


func GetPlayerRects() -> Array[Rect2]:  # on the screen, for the visibility contract
	return player_rects


# The verbs of the bot.
func row() -> void:
	strokes[0] += 1


func point_at(point: Vector2, screen_size: Vector2) -> void:
	hand = HandCursor.reach.position + point / screen_size * HandCursor.reach.size
```

`tests/godot/end2end/test_playthrough.gd`:

```gdscript
extends SceneTree

const FakeVision := preload("res://tests/godot/end2end/fake_vision.gd")
const Contracts := preload("res://tests/godot/end2end/contracts.gd")
const ErrorLog := preload("res://tests/godot/end2end/error_log.gd")

const GAME_HOURS := 5.0
const WALL_LIMIT_MS := 10 * 60 * 1000

const LOG_DIR := "user://tests/logs"

var rng := RandomNumberGenerator.new()
var log_file: FileAccess
var log_path := ""
var fake: Node
var contracts: Contracts
var errors: ErrorLog
var game_seconds := 0.0
var wall_start := Time.get_ticks_msec()


func _initialize() -> void:
	var seed_text := _argument("--seed", str(randi()))
	rng.seed = int(seed_text)
	seed(int(seed_text))  # the game draws from the global generator, too
	_open_log("playthrough", seed_text)
	log_line("SEED: %s (replay with -- --seed=%s)" % [seed_text, seed_text])
	errors = ErrorLog.new(log_line)
	OS.add_logger(errors)
	Engine.time_scale = 10.0
	Engine.max_fps = 0
	fake = FakeVision.new()
	root.add_child(fake)
	root.get_node("/root/Vision").service = fake
	contracts = Contracts.new(root, fake, log_line)
	root.child_entered_tree.connect(func(node: Node) -> void: log_line("SCREEN %s at %.1f s" % [node.name, game_seconds]))
	change_scene_to_file("res://scenes/main/main.tscn")


func _process(delta: float) -> bool:
	game_seconds += delta
	_act()
	contracts.check_frame(delta)
	if errors.count > 0:
		return _finish("engine error: " + errors.first)
	if contracts.stuck_for() > 120.0:
		return _finish("stuck: " + contracts.stuck_reason())
	if Time.get_ticks_msec() - wall_start > WALL_LIMIT_MS:
		return _finish("wall-clock limit")
	if game_seconds > GAME_HOURS * 3600.0:
		return _finish("")
	return false


## Choose like a child: plausible, random; mostly plays, sometimes points, sometimes idles.
func _act() -> void:
	var buttons := contracts.enabled_buttons()
	if not buttons.is_empty():
		fake.hand_tracking = true
		var button: Control = contracts.chosen if contracts.chosen in buttons else buttons[rng.randi() % buttons.size()]
		if button != contracts.chosen:
			log_line("CHOOSE %s (%s) of %d at %.1f s" % [button.get_path(), button.get("text"), buttons.size(), game_seconds])
		contracts.chosen = button
		fake.point_at(button.get_global_rect().get_center(), root.get_visible_rect().size)
		return
	fake.hand_tracking = rng.randf() < 0.02  # a stray hand while playing
	match rng.randi() % 10:
		0:
			fake.steer[0] = rng.randf_range(-1.0, 1.0)
		1:
			fake.brake = not fake.brake
		_:
			if rng.randf() < 0.3:
				fake.row()


func _finish(failure: String) -> bool:
	var broken := contracts.summary()
	log_line("checks: %d, broken: %d, game time: %.0f s" % [contracts.checks, broken.size(), game_seconds])
	for line in broken:
		log_line("  " + line)
	var ok := failure.is_empty() and broken.is_empty()
	if not failure.is_empty():
		log_line("FAILURE: " + failure)
	log_file.close()
	if ok:
		DirAccess.remove_absolute(log_path)  # passed: nothing to look at
		print("RESULT: OK")
	else:
		print("LOG: ", ProjectSettings.globalize_path(log_path))
		print("RESULT: FAILED")
	quit(0 if ok else 1)
	return true


## Always log, so that a failed run never has to be repeated; the log is deleted if the run passes.
func _open_log(test: String, seed_text: String) -> void:
	DirAccess.make_dir_recursive_absolute(LOG_DIR)
	log_path = "%s/%s_%s.log" % [LOG_DIR, test, seed_text]
	log_file = FileAccess.open(log_path, FileAccess.WRITE)


func log_line(text: String) -> void:
	print(text)
	log_file.store_line(text)
	log_file.flush()  # survives a crash


func _argument(name: String, default: String) -> String:
	for argument in OS.get_cmdline_user_args():
		if argument.begins_with(name + "="):
			return argument.substr(name.length() + 1)
	return default
```

Run it with:

```
$GODOT --headless --fixed-fps 60 --path . --script tests/godot/end2end/test_playthrough.gd -- --no-camera --mute --seed=7 --settings=user://tests/playthrough.cfg
```

* Draw expensive things only every 8th frame under test.
* Every Godot test logs this way (a shared `tests/godot/test_log.gd`); `precommit.py` prints the `LOG:` path of a failed test.
* Each test gets its own `--settings=user://tests/<name>.cfg` and writes only below `user://tests/`, so tests run in parallel.

`tests/godot/end2end/error_log.gd` (Godot 4.5+ `Logger`):

```gdscript
extends Logger

const EXPECTED: Array[String] = []  # substrings of errors we accept, each with a reason in a comment

var count := 0
var first := ""
var _log: Callable
var _mutex := Mutex.new()


func _init(a_log: Callable) -> void:
	_log = a_log


func _log_error(
	function: String, file: String, line: int, code: String, rationale: String,
	_editor_notify: bool, _error_type: int, _script_backtraces: Array[ScriptBacktrace]
) -> void:
	var message := "%s (%s:%d in %s): %s" % [rationale, file, line, function, code]
	for expected in EXPECTED:
		if expected in message:
			return
	_mutex.lock()
	count += 1
	if first.is_empty():
		first = message
	_mutex.unlock()
	_log.call_deferred("ENGINE ERROR: " + message)  # may come from another thread


func _log_message(_message: String, _error: bool) -> void:
	pass
```

### 12.5 The contracts

`tests/godot/end2end/contracts.gd`.

* **Every message carries enough to debug without rerunning**: which node (its path), what was measured and what was allowed, the text and its locale, the game time, the screen, the player positions.
  Never "text overflow" alone; always `"%s %s needs %s in %s at size %d (%s)"`.
* Each broken contract is logged **once, with its first example and a count**, so a contract broken on every frame does not drown the log.
* Layout rules are **shared functions**, used by the drawing code, the static layout tests and these contracts.

```gdscript
extends RefCounted

const MIN_FONT_SIZE := 20  # a text that shrinks to fit must stay readable
const OVERLAP_GRACE := HudLayout.GLIDE_SECONDS + 0.8

var checks := 0
var chosen: Control = null
var _root: Window
var _vision: Node
var _log: Callable
var _found := {}
var _covered_for := {}  # HUD path -> seconds it has covered a player
var _placeholder := RegEx.create_from_string("(%d|%s|\\{[a-z_@|]+\\}|<null>)")
var _orphans_at_start := -1


func _init(root: Window, vision: Node, a_log: Callable) -> void:
	_root = root
	_vision = vision
	_log = a_log


func check_frame(delta: float) -> void:
	var screen := _root.get_visible_rect()
	var shown: Array[Control] = []
	for node in _root.find_children("*", "Control", true, false):
		var control := node as Control
		if control.is_visible_in_tree() and control.size != Vector2.ZERO:
			shown.append(control)
	for control in shown:
		_check_inside_screen(control, screen)
		_check_inside_parent(control)
		if control is Label:
			_check_text(control, (control as Label).text, (control as Label).autowrap_mode != TextServer.AUTOWRAP_OFF,
				control.size)
		elif control is Button:
			_check_text(control, (control as Button).text, false, control.size - Vector2(32, 16))
	_check_no_overlaps(shown)
	_check_players_visible(delta)
	_check_one_cursor_on_top()
	_check_no_leaks()
	_check_rules()


## No GUI element overflows the screen (the selection glow is allowed its margin).
func _check_inside_screen(control: Control, screen: Rect2) -> void:
	var rect := control.get_global_rect()
	var allowed := screen.grow(18.0) if control.is_in_group("glow") else screen
	_expect(allowed.encloses(rect), "gui_outside_screen", "%s at %s outside %s" % [control.get_path(), rect, screen])


## No element overflows its container, unless the container clips on purpose.
func _check_inside_parent(control: Control) -> void:
	var parent := control.get_parent() as Control
	if parent == null or parent.clip_contents or control.top_level or control.is_in_group("glow"):
		return
	var rect := control.get_global_rect()
	_expect(parent.get_global_rect().grow(0.5).encloses(rect), "gui_outside_parent",
		"%s at %s outside its parent %s at %s" % [control.get_path(), rect, parent.name, parent.get_global_rect()])


## The text fits its box, is readable, is in the language of the game, and shows no placeholder.
func _check_text(control: Control, raw: String, wraps: bool, box: Vector2) -> void:
	var text := control.atr(raw) if control.auto_translate_mode != Node.AUTO_TRANSLATE_MODE_DISABLED else raw
	var font := control.get_theme_font("font")
	var font_size := control.get_theme_font_size("font_size")
	var width := box.x if wraps else -1.0
	var needed := font.get_multiline_string_size(text, HORIZONTAL_ALIGNMENT_LEFT, width, font_size)
	var locale := TranslationServer.get_locale()
	_expect(needed.x <= box.x + 1.0 and needed.y <= box.y + 1.0, "text_overflow",
		"%s %s needs %s in %s at size %d (%s)" % [control.get_path(), text.c_escape(), needed, box, font_size, locale])
	_expect(font_size >= MIN_FONT_SIZE, "text_too_small", "%s at size %d (%s)" % [control.get_path(), font_size, locale])
	_expect(_placeholder.search(text) == null, "placeholder_shown", "%s shows %s" % [control.get_path(), text])
	for character in text:
		if not font.has_char(character.unicode_at(0)):
			_report("glyph_missing", "%s: %s in %s (%s)" % [control.get_path(), character, text, locale])
			break
	if locale.left(2) != "en" and raw != "" and control.auto_translate_mode != Node.AUTO_TRANSLATE_MODE_DISABLED:
		_expect(text != raw or _is_international(raw), "untranslated", "%s shows %s in %s" % [control.get_path(), raw, locale])


func _is_international(text: String) -> bool:  # numbers, names, symbols
	return not RegEx.create_from_string("[A-Za-z]{3,}").search(text)


## No two buttons overlap, no label overlaps a button (allowed pairs explicit via the group "may_overlap").
func _check_no_overlaps(shown: Array[Control]) -> void:
	var buttons := shown.filter(func(c: Control) -> bool: return c is Button)
	for i in buttons.size():
		var a: Control = buttons[i]
		for j in range(i + 1, buttons.size()):
			var b: Control = buttons[j]
			if a.is_in_group("may_overlap") or b.is_in_group("may_overlap"):
				continue
			_expect(not a.get_global_rect().intersects(b.get_global_rect()), "buttons_overlap",
				"%s and %s" % [a.get_path(), b.get_path()])


## No GUI/HUD element covers a player for longer than the HUD needs to glide away.
func _check_players_visible(delta: float) -> void:
	var players: Array[Rect2] = _vision.GetPlayerRects()
	var screen := _root.get_visible_rect()
	for player in players:
		_expect(screen.encloses(player), "player_off_screen", "player at %s" % player)
	for node in _root.get_tree().get_nodes_in_group("hud"):
		var hud := node as Control
		var key := str(hud.get_path())
		var covers := hud.is_visible_in_tree() and players.any(
			func(p: Rect2) -> bool: return hud.get_global_rect().intersects(p))
		_covered_for[key] = (_covered_for.get(key, 0.0) + delta) if covers else 0.0
		_expect(_covered_for[key] <= OVERLAP_GRACE, "hud_covers_player",
			"%s at %s covers a player at %s" % [key, hud.get_global_rect(), players])


## Exactly one cursor per hand, drawn above every enabled button.
func _check_one_cursor_on_top() -> void:
	var cursors := _root.get_tree().get_nodes_in_group("hand_cursors")
	_expect(cursors.size() <= _vision.GetHands().size() + 1, "extra_cursors", "%d cursors" % cursors.size())


## The tree holds only the autoloads and the current screen; nothing leaks.
func _check_no_leaks() -> void:
	var orphans := int(Performance.get_monitor(Performance.OBJECT_ORPHAN_NODE_COUNT))
	if _orphans_at_start < 0:
		_orphans_at_start = orphans
	_expect(orphans <= _orphans_at_start, "orphan_nodes", "%d orphan nodes" % orphans)


## The rules of the game, one check per invariant: e.g., the boat is on the water and never faster
## than its ship; money >= 0; cargo fits the hold; days never go backwards; stars never decrease;
## at least one button is enabled on every choice screen; the save on disk equals the state in memory
## at each save; a level is done exactly when its end screen shows; the music fits the place.
func _check_rules() -> void:
	pass


func enabled_buttons() -> Array[Control]:
	var result: Array[Control] = []
	for node in _root.get_tree().get_nodes_in_group("dwell_buttons"):
		var button := node as DwellButton
		if button.is_visible_in_tree() and not button.disabled:
			result.append(button)
	return result


func stuck_for() -> float:
	return 0.0  # seconds without progress; per game


func stuck_reason() -> String:
	return ""


func _expect(ok: bool, key: String, example: String) -> void:
	checks += 1
	if not ok:
		_report(key, example)


func _report(key: String, example: String) -> void:
	if not _found.has(key):
		_found[key] = {"count": 0, "example": example}
		var scene := _root.get_tree().current_scene
		_log.call("CONTRACT BROKEN: %s: %s [screen %s, players %s]" % [
			key, example, scene.name if scene != null else "-", _vision.GetPlayerRects()])
	_found[key]["count"] += 1


func summary() -> Array[String]:
	var lines: Array[String] = []
	for key in _found:
		lines.append("%s x%d, first: %s" % [key, _found[key]["count"], _found[key]["example"]])
	return lines
```

Besides the playthrough:

* `test_layout.gd`: every screen and dialog **× every language × the longest names a player can type** (in wide letters, e.g. `"Шолпанайгерім"`) × 1 and 2 players, with `Grammar.longest_crew_forms = true`.
  *Why:* a playthrough sees each screen in only a few languages.
* `test_font.gd`: the font has every character of every `msgstr` of every `.po`.
* `test_hud_fuzz.gd`: the players at random places; after the glide no HUD piece covers them (10.1).
* A context message is listed by Godot as `context\u0004msgid`; remember it when you compare texts with the PO.

### 12.6 Test scenes per mechanic

`scenes/tests/<mechanic>/` holds an isolated level with a script of players (`appear`, `glide`, `chase`, `lose`), at a realistic head speed (900 px/s).
A harness scene (F6 in the editor) runs it with the scripted players, the mouse or the camera; `tests/godot/test_mechanics.gd` replays every one headless and asserts its metrics.
The same scenes serve designing and CI.
`precommit.py` smoke-starts every scene in `scenes/tests/`.
Exclude `scenes/tests/*` from the exports.

### 12.7 Balance bots: is the game balanced, and how long does it take?

A balance bot plays **the rules only** (no rendering, no vision, no screens) with a **strategy**, at a pace that models a child, many times with different seeds, and measures **how long** each milestone takes in play time.
It answers the questions the designers argue about, with numbers:

* How long does a level take?
  How long until the first purchase, the first new ship, the end of a chapter?
* Does any strategy earn much faster than playing as intended (an **exploit**), or get stuck (a **dead end**)?
* Is **cooperation never slower than playing alone**?
* Is the game still finishable by a careless child (random choices)?

Such bots found that the rewards were too small (the first ship after 65 tasks, about 15 hours; the rewards were doubled), an exploit (sleeping again and again instead of rowing earned more than playing), and they keep every level in its target duration.

**Strategies** (each a class with one method, `act`):

| Strategy | Plays like | Finds |
|---|---|---|
| `careful` | a child who does the tasks in order and buys what is needed | the intended duration: the reference for the targets |
| `greedy` | takes the best-paying option every time | rewards that grow too fast; shortcuts |
| `lazy` | does the least that still earns (waits, sleeps, repeats the easiest task) | **exploits**: a lazy strategy must never earn faster than the careful one |
| `random` | chooses uniformly among the allowed actions (a fuzzer of the rules) | dead ends, broken states (with the `assert`s of 12.3) |
| `team` | 2–4 bots of the same strategy together | that cooperation is never slower than playing alone |

The pace models a child, not a machine: e.g. one jump every 1.6 s, 20 strokes per minute, 3 s to choose on a screen, a rest of 30 s between levels.
The bot measures **game time at that pace**, which is the play time of a real child.

`dev/balance/balance.gd` (run: `$GODOT --headless --path . --script dev/balance/balance.gd -- --runs=20 --hours=50 --strategies=careful,lazy,random`):

```gdscript
extends SceneTree
## Play the rules with several strategies and report how long the milestones take.

const DT := 0.1  # s of game time per step
const CHOICE_SECONDS := 3.0  # a child takes this long to choose on a screen


class Strategy:
	extends RefCounted
	var rng: RandomNumberGenerator

	func _init(a_rng: RandomNumberGenerator) -> void:
		rng = a_rng

	## Choose the next action among the allowed ones; the rules carry it out at a child's pace.
	func act(_game: Game) -> Game.Action:
		return null


class Careful:
	extends Strategy

	func act(game: Game) -> Game.Action:
		var task := game.next_task_in_order()
		if task != null:
			return Game.Action.do_task(task)
		var needed := game.cheapest_needed_purchase()
		return Game.Action.buy(needed) if needed != null and game.can_afford(needed) else Game.Action.rest()


class Lazy:
	extends Strategy

	func act(game: Game) -> Game.Action:
		return game.cheapest_earning_action()  # wait, sleep, repeat the easiest task


class Random:
	extends Strategy

	func act(game: Game) -> Game.Action:
		var allowed := game.allowed_actions()
		return allowed[rng.randi() % allowed.size()]


## The play time in hours until each milestone, per strategy and run.
class Report:
	extends RefCounted
	var strategy: String
	var seed: int
	var players: int
	var hours_to: Dictionary = {}  # milestone -> hours
	var stuck := false


func _initialize() -> void:
	var runs := int(_argument("--runs", "20"))
	var max_hours := float(_argument("--hours", "50"))
	var names := _argument("--strategies", "careful,greedy,lazy,random").split(",")
	var reports: Array[Report] = []
	for name in names:
		for players in [1, 2, 4]:
			for run in range(runs):
				reports.append(_play(name, run, players, max_hours))
	_print_table(reports)
	quit(0)


func _play(name: String, seed: int, players: int, max_hours: float) -> Report:
	var rng := RandomNumberGenerator.new()
	rng.seed = seed
	var game := Game.new(players, seed)  # the Godot-free rules, as in the game
	var bots: Array[Strategy] = []
	for i in range(players):
		bots.append(_strategy(name, rng))
	var report := Report.new()
	report.strategy = name
	report.seed = seed
	report.players = players
	var since_progress := 0.0
	while game.time < max_hours * 3600.0 and not game.finished():
		for i in range(players):
			var action := bots[i].act(game)
			game.start(i, action)  # an action takes its time at a child's pace
		game.step(DT)
		for milestone in game.take_milestones():  # "first_purchase", "ship_2", "chapter_1", ...
			if not report.hours_to.has(milestone):
				report.hours_to[milestone] = game.time / 3600.0
			since_progress = 0.0
		since_progress += DT
		if since_progress > 3600.0:  # an hour without progress: a dead end
			report.stuck = true
			break
	return report


func _strategy(name: String, rng: RandomNumberGenerator) -> Strategy:
	match name:
		"careful":
			return Careful.new(rng)
		"lazy":
			return Lazy.new(rng)
		"random":
			return Random.new(rng)
	return Careful.new(rng)


## Print the median and the 10th/90th percentile of the hours to each milestone.
func _print_table(reports: Array[Report]) -> void:
	var groups := {}
	for report in reports:
		var key := "%s x%d" % [report.strategy, report.players]
		for milestone: String in report.hours_to:
			groups.get_or_add(key + " " + milestone, []).append(report.hours_to[milestone])
	for key: String in groups:
		var hours: Array = groups[key]
		hours.sort()
		var at := func(q: float) -> float: return hours[mini(hours.size() - 1, int(q * hours.size()))]
		print("%-40s median %5.1f h  (p10 %5.1f, p90 %5.1f, n=%d)" % [key, at.call(0.5), at.call(0.1), at.call(0.9), hours.size()])
	for report in reports:
		if report.stuck:
			print("STUCK: %s x%d, seed %d, after %s" % [report.strategy, report.players, report.seed, report.hours_to])


func _argument(name: String, default: String) -> String:
	for argument in OS.get_cmdline_user_args():
		if argument.begins_with(name + "="):
			return argument.substr(name.length() + 1)
	return default
```

The report is for the designers, who tune the numbers with it (`--reward-scale=1.5`, `--ship-prices=…` override the data without editing it).
The **targets** go into a test, `tests/godot/test_balance.gd`, with fewer runs, so a change of the data that breaks the balance fails CI:

```gdscript
# The targets of DESIGN.md, checked with 10 seeds each.
_check(median(level_minutes("careful", 1)) >= 4.0 and median(level_minutes("careful", 1)) <= 8.0,
	"a level takes 4 to 8 minutes for one careful player, but took %.1f" % median(level_minutes("careful", 1)))
_check(median(hours_to("careful", 1, "first_ship")) <= 8.0,
	"a careful child gets the first ship within 8 hours, but took %.1f" % median(hours_to("careful", 1, "first_ship")))
_check(median(hours_to("lazy", 1, "first_ship")) >= median(hours_to("careful", 1, "first_ship")),
	"lazy play must not be faster than careful play (an exploit): %.1f h vs %.1f h"
	% [median(hours_to("lazy", 1, "first_ship")), median(hours_to("careful", 1, "first_ship"))])
for players in [2, 4]:
	_check(median(level_minutes("careful", players)) <= 1.1 * median(level_minutes("careful", 1)),
		"%d players must not be slower than one: %.1f vs %.1f minutes"
		% [players, median(level_minutes("careful", players)), median(level_minutes("careful", 1))])
_check(no_run_stuck("random"), "random play got stuck: %s" % stuck_runs("random"))
```

* Write the targets into DESIGN.md first (minutes per level, hours per chapter, the session length of section 20), then let the test hold them.
* The bots play **the same rule classes** as the game (4.1): a balance bot on a copy of the rules measures the copy.
* Re-run the report whenever the data changes (prices, rewards, levels) and paste the table into the pull request.

---

## 13. Saves

* JSON in `user://`, with a `format` number.
* **Atomic**: write `<file>.tmp`, then rename it over the file; never remove the old file first.
  On load, if only the `.tmp` exists, use it.
* **Refuse a newer format** (never overwrite what a newer game wrote).
* **Upgrade older formats** step by step, after copying the original once to `backups/<id>.format<N>.json`.
* **Derive what can be derived** (e.g. the unlocked levels from the finished ones), so a save can never disagree with itself.
* Save the **plan** of a running level, not only its seed, so later data changes do not alter a level in progress.
* `tests/godot/saves/*.json` are **frozen saves of every released format**, including one in the middle of a level; a test loads each and plays on.
  Never change these files: they are what the players have on their disks.

```gdscript
class_name SaveFile
extends RefCounted

const FORMAT := 3


static func write(path: String, data: Dictionary) -> Error:
	data["format"] = FORMAT
	var temporary := path + ".tmp"
	var file := FileAccess.open(temporary, FileAccess.WRITE)
	if file == null:
		return FileAccess.get_open_error()
	file.store_string(JSON.stringify(data, "\t"))
	file.close()
	return DirAccess.rename_absolute(temporary, path)  # replaces the old file in one step


## The save, upgraded to FORMAT; empty if there is none, it is broken, or it is from a newer game.
static func read(path: String, id: String) -> Dictionary:
	var source := path if FileAccess.file_exists(path) else path + ".tmp"
	if not FileAccess.file_exists(source):
		return {}
	var parsed: Variant = JSON.parse_string(FileAccess.get_file_as_string(source))
	if not parsed is Dictionary:
		return {}
	var data: Dictionary = parsed
	var format := int(data.get("format", 0))
	if format > FORMAT:
		return {}
	if format < FORMAT:
		var backups := path.get_base_dir().path_join("backups")
		DirAccess.make_dir_recursive_absolute(backups)
		var backup := backups.path_join("%s.format%d.json" % [id, format])
		if not FileAccess.file_exists(backup):
			DirAccess.copy_absolute(source, backup)
		while format < FORMAT:
			data = _upgrade(data, format)
			format += 1
	return data


static func _upgrade(data: Dictionary, from: int) -> Dictionary:
	match from:
		1:
			pass  # e.g. data["players"] = [data["player"]]; data.erase("player")
		2:
			pass
	data["format"] = from + 1
	return data
```

Ids are the creation time plus a counter: two journeys started within a second get different ids.

---

## 14. Pictures: generative AI on SVG sketches

### 14.1 The pipeline

1. **Sketch in SVG**: generated by a Python script in `dev/pictures/sketches/` or exported from the game (`dev/pictures/export_figures.gd` renders every figure drawn in code at 2x, with a `manifest.json` of sizes and anchors).
   The sketch fixes the layout of the sheet, the sizes, the poses and the anchors.
2. **Prompt**: one Markdown file per sheet, with the same name as the sketch.
   It starts with the **style brief**, sent first in every chat: palette with hex colours, outline colour, "seen from 2–3 m by children aged 6–10", calm and dim backgrounds, bright outlines on everything that moves, a **flat magenta (`#FF00FF`) background**.
3. **Paint** with the image generator from the sketch and the prompt.
   Save the raw output unchanged.
4. **Cut** by script into sprites: key out the magenta, remove a painted fake checkerboard and haze anyway, drop specks and grid lines, crop by alpha, write the anchors.
5. **Fallback**: every figure also has a simple version drawn in code, so a missing picture never breaks the game and the game is playable before the art exists.

**Separate flat folders; one file per sheet; the same name in every folder:**

```
dev/pictures/sketches/dwarves.svg          # the basis (committed)
dev/pictures/prompts/dwarves.md            # the style brief and the prompt (committed)
dev/pictures/prompts/dwarves_redo.md       # the correction prompts, if any (committed)
dev/pictures/painted/dwarves.png           # the raw output of the generator (committed, LFS)
dev/pictures/painted/dwarves_GOOD.png      # a reference of what "right" looks like
assets/pictures/dwarves_walk_1.png         # the cut sprites (LFS)
assets/pictures/dwarves.json               # sizes and anchors of the cut sprites
```

Rules:

* **Commit every prompt and every raw painting.**
  `precommit.py` fails if a painting has no prompt of the same name.
  *Why:* prompts and sheets left in `~/Downloads` were lost to the repository, and the art could not be redone.
* **Recolour with masks** instead of painting variants: an RGB mask per picture (red = skin, green = hair, blue = clothes, white = keep) and a Godot-free `Recoloring.cs` that keeps the shading.
  Generators also paint all clothes in one colour; masks fix that.
* Write the generator's terms on the rights in its output, and an **Altered:** paragraph listing the processing, into THIRD_PARTY_NOTICES.
* Pick **one art pipeline** early (painted or pixel art), and stay with it.
* Public-domain images (paintings, photos from Wikimedia) are fine too: pin each original by SHA-256 in a manifest with its source, license and the alterations, and send a User-Agent and wait between downloads.
* Pixel art: draw SVGs with named palette colours, render them at the shown size, quantize each pixel to the palette colour of most of its 3x3 samples, add the outline; viewport 960x540 with integer scale (640x360 is too coarse for people and ornaments).

### 14.2 Overlays on the player: gray shapes and anchor dots

For pieces the game draws on the player in the camera picture (a helmet, a beard, a cloak anchored to key points), the generator must see **where the body is** and **where each piece is pinned**:

* sketch the player as **neutral gray shapes** (head, torso, arms), flat, without detail, so the generator paints only the pieces;
* mark the **anchors as coloured dots** with their key-point names;
* ask: *"Paint a viking helmet and a braided beard over the dark gray head, centred on the red dot (nose), resting on the blue dots (ears).
  Do not paint the gray body or the dots.
  Flat magenta background."*

The dots tell the painter where things go, but generators move them: **the cutter aligns each piece by its outline or by detected features** (the eyes of a face piece), never by the painted dots.
The anchors the game uses come from the sketch's JSON.
The game scales each piece by the player's body unit and rotates it by the shoulder line; between poses, the pieces follow the head (shoulders fully up and down, half sideways), and each joint has a 1€ filter.

`dev/pictures/sketches/make_overlay_sketch.py`:

```python
#!/usr/bin/env python3
"""
Draw an SVG sketch of a player for painting overlays: the body as gray shapes,
the anchors as dots. Also write the anchors as JSON for the cutter and the game.

Usage: venv/bin/python dev/pictures/sketches/make_overlay_sketch.py <name> [--size 1024]
"""
import argparse
import json
import pathlib
import sys
from dataclasses import dataclass
from typing import Dict, List

REPO_ROOT = pathlib.Path(__file__).resolve().parent.parent.parent.parent
SKETCHES = REPO_ROOT / "dev" / "pictures" / "sketches"

GRAY = "#9a9a9a"
DARK_GRAY = "#6e6e6e"
BACKGROUND = "#ff00ff"  # keyed out by the cutter


@dataclass(frozen=True)
class Point:
    x: float
    y: float


@dataclass(frozen=True)
class Anchor:
    name: str  # the key point (COCO name) the piece is pinned to
    at: Point
    color: str


#: A frontal reference pose in units of the canvas (0..1), COCO key points.
POSE: Dict[str, Point] = {
    "nose": Point(0.50, 0.20),
    "left_eye": Point(0.47, 0.18), "right_eye": Point(0.53, 0.18),
    "left_ear": Point(0.44, 0.19), "right_ear": Point(0.56, 0.19),
    "left_shoulder": Point(0.38, 0.33), "right_shoulder": Point(0.62, 0.33),
    "left_elbow": Point(0.32, 0.48), "right_elbow": Point(0.68, 0.48),
    "left_wrist": Point(0.30, 0.62), "right_wrist": Point(0.70, 0.62),
    "left_hip": Point(0.42, 0.66), "right_hip": Point(0.58, 0.66),
}

ANCHORS: List[Anchor] = [
    Anchor("nose", POSE["nose"], "#e53935"),
    Anchor("left_ear", POSE["left_ear"], "#1e88e5"),
    Anchor("right_ear", POSE["right_ear"], "#1e88e5"),
    Anchor("left_shoulder", POSE["left_shoulder"], "#43a047"),
    Anchor("right_shoulder", POSE["right_shoulder"], "#43a047"),
    Anchor("left_wrist", POSE["left_wrist"], "#fdd835"),
    Anchor("right_wrist", POSE["right_wrist"], "#fdd835"),
]


def limb(a: Point, b: Point, width: float, size: int) -> str:
    return (f'<line x1="{a.x * size:.1f}" y1="{a.y * size:.1f}" x2="{b.x * size:.1f}" y2="{b.y * size:.1f}" '
            f'stroke="{GRAY}" stroke-width="{width * size:.1f}" stroke-linecap="round"/>')


def sketch(size: int) -> str:
    p = POSE
    torso = (p["left_shoulder"], p["right_shoulder"], p["right_hip"], p["left_hip"])
    parts = [
        f'<rect width="{size}" height="{size}" fill="{BACKGROUND}"/>',
        f'<polygon fill="{GRAY}" points="' + " ".join(f"{q.x * size:.1f},{q.y * size:.1f}" for q in torso) + '"/>',
        limb(p["left_shoulder"], p["left_elbow"], 0.06, size),
        limb(p["left_elbow"], p["left_wrist"], 0.05, size),
        limb(p["right_shoulder"], p["right_elbow"], 0.06, size),
        limb(p["right_elbow"], p["right_wrist"], 0.05, size),
        f'<circle cx="{p["nose"].x * size:.1f}" cy="{(p["nose"].y - 0.01) * size:.1f}" '
        f'r="{0.075 * size:.1f}" fill="{DARK_GRAY}"/>',
    ]
    for anchor in ANCHORS:
        x, y = anchor.at.x * size, anchor.at.y * size
        parts.append(f'<circle cx="{x:.1f}" cy="{y:.1f}" r="{0.012 * size:.1f}" fill="{anchor.color}" '
                     f'stroke="white" stroke-width="2"/>')
        parts.append(f'<text x="{x + 0.018 * size:.1f}" y="{y + 0.006 * size:.1f}" font-family="sans-serif" '
                     f'font-size="{0.018 * size:.0f}" fill="white">{anchor.name}</text>')
    body = "\n  ".join(parts)
    return (f'<svg xmlns="http://www.w3.org/2000/svg" width="{size}" height="{size}" '
            f'viewBox="0 0 {size} {size}">\n  {body}\n</svg>\n')


def main() -> int:
    parser = argparse.ArgumentParser(description=__doc__, formatter_class=argparse.RawDescriptionHelpFormatter)
    parser.add_argument("name")
    parser.add_argument("--size", type=int, default=1024)
    args = parser.parse_args()
    SKETCHES.mkdir(parents=True, exist_ok=True)
    (SKETCHES / f"{args.name}.svg").write_text(sketch(args.size), encoding="utf-8")
    anchors = {a.name: {"x": a.at.x, "y": a.at.y} for a in ANCHORS}
    (SKETCHES / f"{args.name}.anchors.json").write_text(json.dumps(anchors, indent=2) + "\n", encoding="utf-8")
    print(f"Wrote {args.name}.svg and {args.name}.anchors.json in {SKETCHES.relative_to(REPO_ROOT)}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

---

## 15. Sound effects and music

* **Licenses: CC0, CC BY 4.0 or public domain.
  Never NC, ND or SA.**
  Credit every author, even of CC0 (THIRD_PARTY_NOTICES table, thanks in the README).
* **One manifest per kind** (`dev/sounds/sources.json`, `dev/music/sources.json`) holds the provenance; the scripts **check the license on the source before every download** and fail otherwise; THIRD_PARTY_NOTICES is generated from the manifests.
* Run the scripts by hand; commit their outputs (Ogg in LFS).
* Temporary files go to `.cache/` in the repository, not `/tmp`: the snap of ffmpeg sees only `$HOME`.

### 15.1 Sound effects

`dev/sounds/sources.json`:

```json
[
  {"name": "jump", "freesound": 412345, "author": "someone", "title": "Boing 3",
   "license": "CC0", "start": 0.12, "end": 0.58},
  {"name": "water", "freesound": 398765, "author": "someone else", "title": "Cave drip",
   "license": "CC BY 4.0", "start": 2.0, "end": 14.0, "loop": true, "filters": ["highpass=f=60"]}
]
```

`dev/sounds/make_sounds.py`:

```python
#!/usr/bin/env python3
"""
Download the sounds of dev/sounds/sources.json from Freesound (checking their licenses),
cut them, and write assets/sounds/<name>.ogg: one-shots faded and peak-normalized,
loops crossfaded at the seam and RMS-normalized.

Usage: venv/bin/python dev/sounds/make_sounds.py [<name> ...]
Needs ffmpeg and sox (sudo apt install ffmpeg sox libsox-fmt-all).
"""
import argparse
import json
import pathlib
import re
import subprocess
import sys
import tempfile
import time
import urllib.error
import urllib.request
from typing import List, Optional

import numpy as np
import pydantic

REPO_ROOT = pathlib.Path(__file__).resolve().parent.parent.parent
SOURCES = REPO_ROOT / "dev" / "sounds" / "sources.json"
SOUNDS_DIR = REPO_ROOT / "assets" / "sounds"
CACHE = REPO_ROOT / ".cache" / "sounds"
USER_AGENT = "dinamika-games-sound-fetcher/1.0"

RATE = 44100
PEAK_DB = -3.0        # one-shots: peak, since loudness needs about 3 s of sound
LOOP_RMS_DB = -20.0   # loops: RMS
CROSSFADE = 0.4       # s, at the seam of a loop
LICENSES = {
    "CC0": {"creativecommons.org/publicdomain/zero/1.0"},
    "CC BY 4.0": {"creativecommons.org/licenses/by/4.0"},
}


class Sound(pydantic.BaseModel):
    model_config = pydantic.ConfigDict(frozen=True, extra="forbid")
    name: str
    freesound: int
    author: str
    title: str
    license: str
    start: float
    end: float
    loop: bool = False
    filters: List[str] = []


def fetch(url: str) -> bytes:
    for attempt in range(5):
        try:
            request = urllib.request.Request(url, headers={"User-Agent": USER_AGENT})
            with urllib.request.urlopen(request, timeout=60) as response:
                return response.read()
        except urllib.error.HTTPError as error:
            if error.code != 429:
                raise
            time.sleep(10 * (attempt + 1))  # Freesound asks us to slow down
    raise RuntimeError(f"Freesound keeps refusing {url}")


def download(sound: Sound) -> pathlib.Path:
    """Download the high-quality preview, after checking the license on the page of the sound."""
    path = CACHE / f"{sound.freesound}.ogg"
    if path.exists():
        return path
    if sound.license not in LICENSES:
        raise RuntimeError(f"{sound.name}: the license {sound.license} is not allowed ({sorted(LICENSES)})")
    page = fetch(f"https://freesound.org/s/{sound.freesound}/").decode("utf-8", "replace")
    found = set(re.findall(r'href="https?://(creativecommons\.org/[^"]+?)/?"', page))
    if found != LICENSES[sound.license]:
        raise RuntimeError(f"{sound.name}: Freesound {sound.freesound} is not under {sound.license}, but {found}")
    preview: Optional[re.Match] = re.search(r'data-ogg="([^"]+)"', page)
    if preview is None:
        raise RuntimeError(f"{sound.name}: found no preview on the page of Freesound {sound.freesound}")
    CACHE.mkdir(parents=True, exist_ok=True)
    path.write_bytes(fetch(preview.group(1).replace("-lq.ogg", "-hq.ogg")))
    time.sleep(2.0)
    return path


def decode(sound: Sound, source: pathlib.Path) -> np.ndarray:
    filters = [f"atrim=start={sound.start}:end={sound.end}", "asetpts=PTS-STARTPTS", *sound.filters]
    raw = subprocess.run(
        ["ffmpeg", "-loglevel", "error", "-i", str(source), "-ac", "1", "-ar", str(RATE),
         "-af", ",".join(filters), "-f", "f32le", "-"],
        check=True, stdout=subprocess.PIPE).stdout
    return np.frombuffer(raw, dtype=np.float32).astype(np.float64)


def one_shot(samples: np.ndarray) -> np.ndarray:
    fade_in, fade_out = int(0.005 * RATE), int(0.08 * RATE)
    samples = samples.copy()
    samples[:fade_in] *= np.linspace(0.0, 1.0, fade_in)
    samples[-fade_out:] *= np.linspace(1.0, 0.0, fade_out)
    return samples * (10.0 ** (PEAK_DB / 20.0) / max(np.abs(samples).max(), 1e-9))


def loop(samples: np.ndarray) -> np.ndarray:
    overlap = int(CROSSFADE * RATE)
    length = len(samples) - overlap
    if length <= overlap:
        raise RuntimeError("A loop must be longer than twice the crossfade.")
    result = samples[:length].copy()
    phase = 0.5 * np.pi * np.arange(overlap) / overlap
    # Equal power, since the two ends are not correlated.
    result[:overlap] = samples[:overlap] * np.sin(phase) + samples[length:] * np.cos(phase)
    rms = np.sqrt(np.mean(result**2))
    result *= 10.0 ** (LOOP_RMS_DB / 20.0) / max(rms, 1e-9)
    peak = np.abs(result).max()
    return result * (0.99 / peak) if peak > 0.99 else result


def encode(samples: np.ndarray, target: pathlib.Path) -> None:
    subprocess.run(["sox", "-t", "f32", "-c", "1", "-r", str(RATE), "-", "-C", "5", str(target)],
                   check=True, input=samples.astype(np.float32).tobytes())


def main() -> int:
    parser = argparse.ArgumentParser(description=__doc__, formatter_class=argparse.RawDescriptionHelpFormatter)
    parser.add_argument("names", nargs="*", help="Only these sounds (default: all).")
    args = parser.parse_args()
    sounds = [Sound(**entry) for entry in json.loads(SOURCES.read_text(encoding="utf-8"))]
    SOUNDS_DIR.mkdir(parents=True, exist_ok=True)
    try:
        for sound in sounds:
            if args.names and sound.name not in args.names:
                continue
            samples = decode(sound, download(sound))
            target = SOUNDS_DIR / f"{sound.name}.ogg"
            encode(loop(samples) if sound.loop else one_shot(samples), target)
            print(f"Wrote {target.relative_to(REPO_ROOT)}")
    except (RuntimeError, subprocess.CalledProcessError) as exception:
        print(str(exception), file=sys.stderr)
        return 1
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

At runtime (`scenes/audio/audio.gd`, autoload `Audio`): `Audio.play("jump")`, `Audio.music("lakes")`, `Audio.ambient("water")`; a table of gains per sound, a random pitch variation, several takes per sound, at most 10 voices, one looping ambience, `--mute`, and a `played` counter the end-to-end test asserts on.
Keep the streams you play out of the resource cache (`ResourceLoader.CACHE_MODE_IGNORE`), so the audio server holding one at exit is not reported as a leak.

### 15.2 Music: MIDI rendered with FluidSynth

**Music is MIDI**: the source is a `.mid` file, rendered once by hand with **FluidSynth** and the **FluidR3 GM** soundfont, loudness-normalized to Ogg with ffmpeg.
Both the `.mid` and the `.ogg` are committed; the game plays only the Ogg, and never needs FluidSynth.
*Why:* about 150 lines of script, one dependency per step, clear licenses, and a sound that is good enough; a synthesizer of our own sounds nicer but costs hundreds of lines per game.

Install: `sudo apt install fluidsynth fluid-soundfont-gm ffmpeg` (macOS: `brew install fluid-synth ffmpeg`, and download `FluidR3_GM.sf2`; pass it with `--soundfont`).

**Where the MIDI comes from** (only CC0 or public domain):

* Wikimedia Commons, category *MIDI files* (e.g., the traditional tunes of Rabanus Flavus, CC0; Peter Gerloff, CC0): the license is checked through the API.
* Mutopia (public domain): the page must say "Public Domain".
* Our own MIDI or ABC transcriptions of public-domain melodies (convert ABC with `abc2midi`).

`dev/music/sources.json`, one entry per song; the slug is the file name, and the mood tells the game when to play it:

```json
[
  {
    "slug": "lakes_1",
    "mood": "lakes",
    "title": "Herr Mannelig",
    "credit": "Rabanus Flavus",
    "license": "CC0",
    "commons": "File:Herr Mannelig.mid",
    "sha256": "…",
    "gain": 1.0
  },
  {
    "slug": "party_1",
    "mood": "party",
    "title": "Our own transcription of a traditional dance",
    "credit": "Dinamika Games",
    "license": "MIT",
    "file": "dev/music/own/party_1.mid"
  }
]
```

`dev/music/make_music.py`:

```python
#!/usr/bin/env python3
"""
Fetch the MIDI files of dev/music/sources.json (checking their licenses) into
assets/music/<slug>.mid, and render them to assets/music/<slug>.ogg with FluidSynth.

Usage: venv/bin/python dev/music/make_music.py [<slug> ...] [--soundfont PATH] [--force]

Needs fluidsynth, the FluidR3 GM soundfont and ffmpeg
(sudo apt install fluidsynth fluid-soundfont-gm ffmpeg).
"""
import argparse
import hashlib
import json
import pathlib
import shutil
import subprocess
import sys
import tempfile
import urllib.parse
import urllib.request
from typing import List, Optional

import pydantic

REPO_ROOT = pathlib.Path(__file__).resolve().parent.parent.parent
SOURCES = REPO_ROOT / "dev" / "music" / "sources.json"
MUSIC_DIR = REPO_ROOT / "assets" / "music"
CACHE = REPO_ROOT / ".cache"
DEFAULT_SOUNDFONT = pathlib.Path("/usr/share/sounds/sf2/FluidR3_GM.sf2")

RATE = 44100
LOUDNESS = -20.0       # LUFS, the same for every song
TRUE_PEAK = -1.5       # dBTP
OGG_QUALITY = 4
ALLOWED_LICENSES = {"CC0", "Public domain", "MIT"}
USER_AGENT = "dinamika-games-music-fetcher/1.0"


class Song(pydantic.BaseModel):
    model_config = pydantic.ConfigDict(frozen=True, extra="forbid")
    slug: str
    mood: str
    title: str
    credit: str
    license: str
    commons: Optional[str] = None   # "File:....mid" on Wikimedia Commons
    url: Optional[str] = None       # a direct download (e.g., Mutopia); the page must be checked by hand
    file: Optional[str] = None      # our own file, relative to the repository
    sha256: Optional[str] = None    # pins a downloaded file: a changed upload fails loudly
    gain: float = 1.0               # FluidSynth's gain, if a song clips or is too quiet before normalizing


def fetch(url: str) -> bytes:
    request = urllib.request.Request(url, headers={"User-Agent": USER_AGENT})
    with urllib.request.urlopen(request, timeout=60) as response:
        return response.read()


def commons_url(title: str, expected_license: str) -> str:
    """Find the download URL on Wikimedia Commons, and fail unless the license is as expected."""
    query = urllib.parse.urlencode({"action": "query", "prop": "imageinfo", "iiprop": "url|extmetadata",
                                    "format": "json", "titles": title})
    answer = json.loads(fetch(f"https://commons.wikimedia.org/w/api.php?{query}"))
    info = next(iter(answer["query"]["pages"].values()))["imageinfo"][0]
    found = info["extmetadata"]["LicenseShortName"]["value"]
    if found.lower() != expected_license.lower():
        raise RuntimeError(f"{title} is not under {expected_license}, but {found}")
    return info["url"]


def obtain_midi(song: Song, force: bool) -> pathlib.Path:
    target = MUSIC_DIR / f"{song.slug}.mid"
    if target.exists() and not force:
        return target
    if song.license not in ALLOWED_LICENSES:
        raise RuntimeError(f"{song.slug}: the license {song.license} is not allowed ({sorted(ALLOWED_LICENSES)})")
    if song.file is not None:
        shutil.copy2(REPO_ROOT / song.file, target)
    else:
        if song.commons is not None:
            data = fetch(commons_url(song.commons, song.license))
        elif song.url is not None:
            data = fetch(song.url)
        else:
            raise RuntimeError(f"{song.slug}: give one of commons, url or file")
        digest = hashlib.sha256(data).hexdigest()
        if song.sha256 is None:
            raise RuntimeError(f"{song.slug}: pin the download with \"sha256\": \"{digest}\"")
        if digest != song.sha256:
            raise RuntimeError(f"{song.slug}: the download changed: sha256 {digest}, expected {song.sha256}")
        target.write_bytes(data)
    print(f"Fetched {target.relative_to(REPO_ROOT)}")
    return target


def render(song: Song, midi: pathlib.Path, soundfont: pathlib.Path) -> pathlib.Path:
    target = MUSIC_DIR / f"{song.slug}.ogg"
    CACHE.mkdir(exist_ok=True)
    with tempfile.TemporaryDirectory(dir=CACHE) as tmp:  # not /tmp: the ffmpeg snap sees only $HOME
        wav = pathlib.Path(tmp) / "song.wav"
        command = ["fluidsynth", "-ni", "-q", "-r", str(RATE), "-g", str(0.5 * song.gain), "-F", str(wav)]
        command += [str(soundfont), str(midi)]
        subprocess.run(command, check=True)
        subprocess.run(
            ["ffmpeg", "-y", "-loglevel", "error", "-i", str(wav), "-af",
             # Trim the silence at the end (FluidSynth renders the reverb tail), then normalize the loudness.
             "silenceremove=stop_periods=-1:stop_duration=1.5:stop_threshold=-60dB,"
             f"loudnorm=I={LOUDNESS}:TP={TRUE_PEAK}",
             "-ar", str(RATE), "-c:a", "libvorbis", "-q:a", str(OGG_QUALITY), str(target)],
            check=True,
        )
    print(f"Rendered {target.relative_to(REPO_ROOT)}")
    return target


def main() -> int:
    parser = argparse.ArgumentParser(description=__doc__, formatter_class=argparse.RawDescriptionHelpFormatter)
    parser.add_argument("slugs", nargs="*", help="Only these songs (default: all).")
    parser.add_argument("--soundfont", type=pathlib.Path, default=DEFAULT_SOUNDFONT)
    parser.add_argument("--force", action="store_true", help="Fetch the MIDI files again.")
    args = parser.parse_args()

    for tool in ("fluidsynth", "ffmpeg"):
        if shutil.which(tool) is None:
            print(f"{tool} is missing; see the docstring.", file=sys.stderr)
            return 1
    if not args.soundfont.is_file():
        print(f"The soundfont {args.soundfont} is missing; pass --soundfont.", file=sys.stderr)
        return 1

    songs: List[Song] = [Song(**entry) for entry in json.loads(SOURCES.read_text(encoding="utf-8"))]
    unknown = set(args.slugs) - {song.slug for song in songs}
    if unknown:
        print(f"Unknown songs: {sorted(unknown)}", file=sys.stderr)
        return 1
    MUSIC_DIR.mkdir(parents=True, exist_ok=True)
    try:
        for song in songs:
            if args.slugs and song.slug not in args.slugs:
                continue
            render(song, obtain_midi(song, args.force), args.soundfont)
    except (RuntimeError, subprocess.CalledProcessError) as exception:
        print(str(exception), file=sys.stderr)
        return 1
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

Rules:

* **One loudness for all songs** (-20 LUFS) and the sound effects peak-normalized to -3 dBFS; the game then mixes the music at -12 dB under the effects.
  No per-song volume fiddling in the game.
* **Credit** every song and the soundfont ("Rendered with FluidSynth and the FluidR3 GM SoundFont by Frank Wen, MIT") in THIRD_PARTY_NOTICES.md; generate the table from `sources.json`.
* **Do not vendor the soundfont** (140 MB); it is installed with the tools.
  The rendered Ogg files are committed (LFS), so the game never needs FluidSynth.
* Change the sound of a song by editing the MIDI (instruments are General MIDI program changes; any MIDI editor, or `mido` in Python), not by post-processing the Ogg.

At runtime: music by mood (start, farm, lakes, party, chase), shuffled, crossfaded over 1.5 s, at -12 dB under the effects.

---

## 16. Export and `dev/publish.py`

**Never export from the editor; export with `dev/publish.py`.**
Godot's export exits with 0 even when the C# build failed (the package then lacks the C# part), it does not know that the Windows package needs the camera extension next to the `.exe`, and it does not collect the licenses.

`export_presets.cfg` (the filters are the same in all three presets):

```ini
[preset.0]
name="Windows"
platform="Windows Desktop"
export_filter="all_resources"
include_filter="*.json, *.onnx, *.po"
exclude_filter="tests/*, dev/*, venv/*, scenes/tests/*"
export_path="build/windows/<game>.exe"
[preset.0.options]
binary_format/architecture="x86_64"
codesign/enable=false

[preset.1]
name="Linux"
platform="Linux"
include_filter="*.json, *.onnx, *.po"
exclude_filter="tests/*, dev/*, venv/*, scenes/tests/*"
export_path="build/linux/<game>.x86_64"
[preset.1.options]
binary_format/architecture="x86_64"
binary_format/embed_pck=false

; preset.2 = macOS: the same filters, plus the options of section 5.6
```

`dev/publish.py`:

```python
#!/usr/bin/env python3
"""
Export the game for Windows, Linux and macOS, check the packages, add the licenses,
zip them into dist/, and optionally upload them to a GitHub release.

Usage:
    venv/bin/python dev/publish.py [--platforms linux windows macos] [--godot PATH]
    venv/bin/python dev/publish.py --tag v1.2.0 --upload     # in CI, on a release

Godot exits with 0 even when the export fails (e.g., the C# build), so we scan its
output for errors and check that every package has the C# part (ONNX Runtime).
"""
import argparse
import os
import pathlib
import re
import shutil
import subprocess
import sys
import zipfile
from dataclasses import dataclass
from typing import Dict, List, Optional

REPO_ROOT = pathlib.Path(os.path.realpath(__file__)).parent.parent
BUILD_DIR = REPO_ROOT / "build"
DIST_DIR = REPO_ROOT / "dist"
EXTENSION_DIR = REPO_ROOT / "addons" / "CameraServerExtension"

SLUG = "my-game"            # the file names of the packages
ASSEMBLY = "MyGame"         # the name of the .csproj; Godot names data_<ASSEMBLY>_<platform>_<arch>
MODELS = ["rtmo", "rtmpose", "yolox"]
HARMLESS_ERRORS = re.compile(r"leaked|still in use|Pages in use|Unreferenced static")


@dataclass(frozen=True)
class Platform:
    preset: str
    output: str              # relative to build/; macOS exports a zip, the others a folder
    onnx_runtime: List[str]  # the native libraries every package must have


PLATFORMS: Dict[str, Platform] = {
    "linux": Platform("Linux", f"linux/{SLUG}.x86_64",
                      [f"data_{ASSEMBLY}_linuxbsd_x86_64/libonnxruntime.so"]),
    "windows": Platform("Windows", f"windows/{SLUG}.exe",
                        [f"data_{ASSEMBLY}_windows_x86_64/onnxruntime.dll"]),
    "macos": Platform("macOS", f"macos/{SLUG}.zip",
                      [f"data_{ASSEMBLY}_macos_arm64/libonnxruntime.dylib",
                       f"data_{ASSEMBLY}_macos_x86_64/libonnxruntime.dylib"]),
}


def find_godot(explicit: Optional[str]) -> Optional[str]:
    if explicit is not None:
        return explicit
    if os.environ.get("GODOT"):
        return os.environ["GODOT"]
    for name in ("godot", "godot4", "godot-mono", "godot4-mono"):
        found = shutil.which(name)
        if found is not None:
            return found
    return None


def run(cmd: List[str]) -> None:
    print(f"Executing: {' '.join(cmd)}", flush=True)
    subprocess.run(cmd, cwd=REPO_ROOT, check=True)


def project_version() -> str:
    text = (REPO_ROOT / "project.godot").read_text(encoding="utf-8")
    match = re.search(r'^config/version="([^"]+)"', text, re.MULTILINE)
    if match is None:
        raise RuntimeError("project.godot has no config/version.")
    return match.group(1)


def check_versions(tag: Optional[str]) -> str:
    """The tag, project.godot and the macOS preset must agree; the tests use the game's ONNX Runtime."""
    version = project_version()
    if tag is not None and tag.removeprefix("v") != version:
        raise RuntimeError(f"The tag {tag} does not match config/version={version} in project.godot.")
    presets = (REPO_ROOT / "export_presets.cfg").read_text(encoding="utf-8")
    for key in ("application/short_version", "application/version"):
        match = re.search(rf'^{re.escape(key)}="([^"]*)"', presets, re.MULTILINE)
        if match is not None and match.group(1) != version:
            raise RuntimeError(f"The macOS preset has {key}={match.group(1)}, expected {version}.")
    onnx = set(re.findall(r'Include="Microsoft\.ML\.OnnxRuntime" Version="([^"]+)"',
                          "".join(p.read_text(encoding="utf-8") for p in REPO_ROOT.rglob("*.csproj")
                                  if "venv" not in p.parts)))
    if len(onnx) != 1:
        raise RuntimeError(f"Expected one ONNX Runtime version across the projects, got {sorted(onnx)}.")
    return version


def export(godot: str, preset: str, output: pathlib.Path) -> None:
    cmd = [godot, "--headless", "--path", ".", "--export-release", preset, str(output)]
    print(f"Executing: {' '.join(cmd)}", flush=True)
    completed = subprocess.run(cmd, cwd=REPO_ROOT, stdout=subprocess.PIPE, stderr=subprocess.STDOUT,
                               text=True, check=True)
    print(completed.stdout, flush=True)
    errors = [line for line in completed.stdout.splitlines()
              if re.match(r"^(ERROR|SCRIPT ERROR)", line) and not HARMLESS_ERRORS.search(line)]
    if errors:
        raise RuntimeError(f"The export for {preset} reported errors:\n" + "\n".join(errors))


def check_onnx_runtime(name: str, platform: Platform, output: pathlib.Path) -> None:
    if name == "macos":
        with zipfile.ZipFile(output) as archive:
            names = archive.namelist()
        missing = [lib for lib in platform.onnx_runtime if not any(n.endswith(lib) for n in names)]
    else:
        missing = [lib for lib in platform.onnx_runtime if not (output.parent / lib).is_file()]
    if missing:
        raise RuntimeError(f"The {name} package misses {missing}: the C# part was not built.")


def onnx_runtime_package() -> pathlib.Path:
    project = (REPO_ROOT / f"{ASSEMBLY}.csproj").read_text(encoding="utf-8")
    match = re.search(r'Include="Microsoft\.ML\.OnnxRuntime" Version="([^"]+)"', project)
    if match is None:
        raise RuntimeError(f"{ASSEMBLY}.csproj references no ONNX Runtime.")
    packages = pathlib.Path(os.environ.get("NUGET_PACKAGES", pathlib.Path.home() / ".nuget" / "packages"))
    return packages / "microsoft.ml.onnxruntime" / match.group(1)


def notices() -> Dict[str, pathlib.Path]:
    """Map each license file of a package (its path in the package) to its source."""
    result = {"LICENSE": REPO_ROOT / "LICENSE", "THIRD_PARTY_NOTICES.md": REPO_ROOT / "THIRD_PARTY_NOTICES.md"}
    for path in sorted((REPO_ROOT / "licenses").rglob("*")):
        if path.is_file() and path.name != ".gdignore":
            result[path.relative_to(REPO_ROOT).as_posix()] = path
    for model in MODELS:
        result[f"licenses/models/{model}-LICENSE"] = REPO_ROOT / "scenes" / "vision" / model / "model" / "LICENSE"
    result["licenses/onnxruntime/LICENSE"] = onnx_runtime_package() / "LICENSE"
    result["licenses/onnxruntime/ThirdPartyNotices.txt"] = onnx_runtime_package() / "ThirdPartyNotices.txt"
    for name, source in result.items():
        if not source.is_file():
            raise RuntimeError(f"The license {name} is missing: {source}")
    return result


def copy_licenses(directory: pathlib.Path) -> None:
    for name, source in notices().items():
        target = directory / name
        target.parent.mkdir(parents=True, exist_ok=True)
        shutil.copy2(source, target)


def add_camera_extension(directory: pathlib.Path) -> None:
    """Godot 4.6 has no camera on Windows: put the extension next to the executable."""
    for name in ("CameraServerExtension.gdextension", "libcameraserver-extension.windows.dll"):
        shutil.copy2(EXTENSION_DIR / name, directory / name)
    license_dir = directory / "licenses" / "camera-server-extension"
    license_dir.mkdir(parents=True, exist_ok=True)
    shutil.copy2(EXTENSION_DIR / "LICENSE", license_dir / "LICENSE")


def add_licenses_to_archive(archive: pathlib.Path) -> None:
    with zipfile.ZipFile(archive, "a", compression=zipfile.ZIP_DEFLATED) as zip_file:
        existing = set(zip_file.namelist())
        for name, source in notices().items():
            if name not in existing:
                zip_file.write(source, name)


def zip_package(name: str, output: pathlib.Path, version: str) -> pathlib.Path:
    DIST_DIR.mkdir(exist_ok=True)
    target = DIST_DIR / f"{SLUG}-{version}-{name}.zip"
    if name == "macos":
        shutil.copy2(output, target)
        return target
    with zipfile.ZipFile(target, "w", compression=zipfile.ZIP_DEFLATED) as archive:
        for path in sorted(output.parent.rglob("*")):
            if path.is_file():
                info = zipfile.ZipInfo.from_file(path, f"{SLUG}/{path.relative_to(output.parent).as_posix()}")
                if path == output or path.suffix in (".x86_64", ".sh"):
                    info.external_attr = 0o755 << 16  # keep the executable bit for Linux
                with path.open("rb") as source:
                    archive.writestr(info, source.read(), compress_type=zipfile.ZIP_DEFLATED)
    return target


def main() -> int:
    parser = argparse.ArgumentParser(description=__doc__, formatter_class=argparse.RawDescriptionHelpFormatter)
    parser.add_argument("--platforms", nargs="+", choices=sorted(PLATFORMS), default=sorted(PLATFORMS))
    parser.add_argument("--godot", help="The Godot .NET binary; else $GODOT or the PATH.")
    parser.add_argument("--tag", help="The release tag, e.g. v1.2.0; must match project.godot.")
    parser.add_argument("--upload", action="store_true", help="Upload the zips to the release --tag with gh.")
    args = parser.parse_args()

    godot = find_godot(args.godot)
    if godot is None:
        print("Could not find Godot; set GODOT or pass --godot.", file=sys.stderr)
        return 1
    if args.upload and args.tag is None:
        print("--upload needs --tag.", file=sys.stderr)
        return 1

    # The .gdignore hides the packages from Godot, which would otherwise import them and try
    # to load the camera extension of the Windows package on every platform.
    for directory in (BUILD_DIR, DIST_DIR):
        directory.mkdir(exist_ok=True)
        (directory / ".gdignore").touch()

    try:
        version = check_versions(args.tag)
        run(["dotnet", "build", f"{ASSEMBLY}.sln", "-c", "Release"])
        zips = []
        for name in args.platforms:
            platform = PLATFORMS[name]
            output = BUILD_DIR / platform.output
            if output.parent.exists():
                shutil.rmtree(output.parent)
            output.parent.mkdir(parents=True)
            export(godot, platform.preset, output)
            check_onnx_runtime(name, platform, output)
            if name == "macos":
                add_licenses_to_archive(output)
            else:
                copy_licenses(output.parent)
            if name == "windows":
                add_camera_extension(output.parent)
            zips.append(zip_package(name, output, version))
            print(f"Packaged {platform.preset} into {zips[-1].relative_to(REPO_ROOT)}.", flush=True)
        if args.upload:
            run(["gh", "release", "upload", args.tag, "--clobber", *map(str, zips)])
    except subprocess.CalledProcessError as exception:
        print(f"Failed with exit code {exception.returncode}.", file=sys.stderr)
        return 1
    except RuntimeError as exception:
        print(str(exception), file=sys.stderr)
        return 1
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

**Releasing**: bump `config/version` in `project.godot` and `application/short_version`/`version` of the macOS preset (numbers only), then publish a GitHub release tagged `v<version>`; the workflow does the rest.
The macOS app is ad-hoc signed and not notarized (right-click → Open the first time); Windows is unsigned (SmartScreen: More info → Run anyway).
The README says both.
Sign and notarize once the game goes public.

---

## 17. GitHub Actions

`.github/workflows/ci.yaml`: on every push to `main` and every pull request; formatting and lint first as a fast fail, then everything else, through the same `precommit.py` as locally.

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

env:
  GODOT_VERSION: "4.6.2"

jobs:
  check:
    runs-on: ubuntu-latest
    timeout-minutes: 60
    steps:
      - uses: actions/checkout@v4
        with:
          lfs: true

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: "8.0.x"

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Cache Godot
        id: cache-godot
        uses: actions/cache@v4
        with:
          path: ~/godot
          key: godot-${{ env.GODOT_VERSION }}-mono-linux

      - name: Download Godot
        if: steps.cache-godot.outputs.cache-hit != 'true'
        run: |
          mkdir -p ~/godot
          cd ~/godot
          curl -sSLO "https://github.com/godotengine/godot/releases/download/${GODOT_VERSION}-stable/Godot_v${GODOT_VERSION}-stable_mono_linux_x86_64.zip"
          unzip -q "Godot_v${GODOT_VERSION}-stable_mono_linux_x86_64.zip"
          rm "Godot_v${GODOT_VERSION}-stable_mono_linux_x86_64.zip"

      - name: Set GODOT
        run: echo "GODOT=$HOME/godot/Godot_v${GODOT_VERSION}-stable_mono_linux_x86_64/Godot_v${GODOT_VERSION}-stable_mono_linux.x86_64" >> "$GITHUB_ENV"

      - name: Install the development tools
        run: |
          python -m venv venv
          touch venv/.gdignore
          venv/bin/pip install -r dev/requirements.txt

      - name: Restore
        run: dotnet restore

      # NOTE (mristin): We check the formatting and lint first, as a fast fail.
      - name: Check the formatting and lint
        run: venv/bin/python dev/continuous_integration/precommit.py --select reformat lint

      # NOTE (mristin): While prototyping, add --prototype; drop it once the game works (section 1).
      - name: Run the checks and the tests
        run: venv/bin/python dev/continuous_integration/precommit.py --skip reformat lint
```

`.github/workflows/publish.yaml`: on a published release, or by hand (without a tag: a dry run that only uploads the packages as artifacts).

```yaml
name: Publish

on:
  release:
    types: [published]
  workflow_dispatch:
    inputs:
      tag:
        description: "The release tag to upload to (empty: a dry run that only uploads artifacts)"
        required: false

env:
  GODOT_VERSION: "4.6.2"

permissions:
  contents: write

jobs:
  publish:
    runs-on: ubuntu-latest
    timeout-minutes: 60
    steps:
      - uses: actions/checkout@v4
        with:
          lfs: true

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: "8.0.x"

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      # NOTE (mristin): Do not call the variable VERSION! MSBuild picks up the environment
      # variables as properties (case-insensitively), so VERSION would override the project
      # version, and dotnet restore fails on tags such as v1.0.0 that are not valid NuGet versions.
      - name: Determine the tag
        run: |
          tag="${{ github.event.release.tag_name || inputs.tag }}"
          echo "RELEASE_TAG=${tag}" >> "$GITHUB_ENV"

      - name: Install Godot and the export templates
        run: |
          mkdir -p ~/godot && cd ~/godot
          base="https://github.com/godotengine/godot/releases/download/${GODOT_VERSION}-stable"
          curl -sSLO "${base}/Godot_v${GODOT_VERSION}-stable_mono_linux_x86_64.zip"
          unzip -q "Godot_v${GODOT_VERSION}-stable_mono_linux_x86_64.zip"
          curl -sSLO "${base}/Godot_v${GODOT_VERSION}-stable_mono_export_templates.tpz"
          templates="$HOME/.local/share/godot/export_templates/${GODOT_VERSION}.stable.mono"
          mkdir -p "$templates"
          unzip -q "Godot_v${GODOT_VERSION}-stable_mono_export_templates.tpz"   # a .tpz is a zip
          mv templates/* "$templates/"
          echo "GODOT=$HOME/godot/Godot_v${GODOT_VERSION}-stable_mono_linux_x86_64/Godot_v${GODOT_VERSION}-stable_mono_linux.x86_64" >> "$GITHUB_ENV"

      - name: Install the development tools
        run: |
          python -m venv venv
          touch venv/.gdignore
          venv/bin/pip install -r dev/requirements.txt

      - name: Import
        run: |
          dotnet restore
          "$GODOT" --headless --path . --import

      - name: Export, check and package
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          if [ -n "$RELEASE_TAG" ]; then
            venv/bin/python dev/publish.py --tag "$RELEASE_TAG" --upload
          else
            venv/bin/python dev/publish.py
          fi

      - uses: actions/upload-artifact@v4
        with:
          name: packages
          path: dist/*.zip
```

Every run with `lfs: true` downloads all LFS files and counts against the bandwidth quota.
Check out without LFS and restore `.git/lfs` from a cache instead:

```yaml
      - uses: actions/checkout@v4   # without lfs: true
      - name: List the LFS files
        run: git lfs ls-files -l | cut -d' ' -f1 | sort > .lfs-assets-id
      - uses: actions/cache@v4
        with:
          path: .git/lfs
          key: lfs-${{ hashFiles('.lfs-assets-id') }}
      - name: Pull the LFS files
        run: git lfs pull
```

---

## 18. Development scripts in `dev/`

**Every development script lives in `dev/` and is Python** (3.10+); only what needs the engine is a Godot script (`$GODOT --headless --script dev/...`).
No shell scripts beyond one-liners.

Conventions:

* a module docstring with the usage line; `argparse.ArgumentParser(description=__doc__)`;
* `REPO_ROOT = pathlib.Path(__file__).resolve().parent.parent` (one `.parent` more per folder level);
* records as frozen pydantic models or dataclasses (4.2);
* `def main() -> int` and `sys.exit(main())`; a failure prints what is wrong and how to fix it, and returns 1;
* black (line length 100) formats `dev/`; one `venv/` with the pinned `dev/requirements.txt` and a `.gdignore`; heavy dependencies used rarely (training) go into a second `venv-ml/` that CI never installs;
* a tool that needs the game's logic calls the game's code (a `dotnet run` project in `dev/`, or a Godot script); never a Python copy of it.

`dev/requirements.txt`:

```
black==24.10.0
gdtoolkit==4.5.0
numpy==2.2.6
pillow==11.0.0
pydantic==2.14.4
polib==1.2.0
onnx==1.17.0
opencv-python-headless==4.11.0.86
```

The scripts every game has:

| Script | Does |
|---|---|
| `dev/continuous_integration/precommit.py` | every check, in named steps; the same locally and in CI |
| `dev/publish.py` | export, check, license, zip, upload (16) |
| `dev/write_godot_licenses.gd` | `licenses/godot/` from the engine (19.4) |
| `dev/translations/extract.py`, `check.py` | POT/PO (11.5) |
| `dev/pictures/sketches/make_*.py`, `dev/pictures/cut_*.py` | sketches, cutting the paintings (14) |
| `dev/sounds/make_sounds.py`, `dev/music/make_music.py` | sound effects and music (15) |
| `dev/vision/record.gd`, `dev/vision/analyze.py` | consented recordings and their analysis (6.8) |
| `dev/balance/balance.gd` | the economy bots (12.7) |
| `dev/screenshots/take_screenshots.py` | every screen in every language on Xvfb, for the README and for review |

`dev/continuous_integration/precommit.py`:

```python
#!/usr/bin/env python3
"""Run the checks before a commit; CI runs the same. Steps: see STEPS."""
import argparse
import concurrent.futures
import os
import pathlib
import re
import subprocess
import sys
from typing import Callable, Dict, List, Tuple

REPO_ROOT = pathlib.Path(__file__).resolve().parent.parent.parent
SOLUTION = next(REPO_ROOT.glob("*.sln")).name

# Godot reports some errors only on the output, but still exits with zero.
# Any error fails: fix leaks at exit instead of filtering them.
GODOT_ERROR_RE = re.compile(r"^(SCRIPT ERROR|ERROR|Parse Error)", re.MULTILINE)


def run(cmd: List[str], timeout: float = 1800.0) -> Tuple[int, str]:
    completed = subprocess.run(cmd, cwd=REPO_ROOT, stdout=subprocess.PIPE, stderr=subprocess.STDOUT,
                               text=True, timeout=timeout)
    return completed.returncode, completed.stdout


def godot_failed(code: int, output: str) -> bool:
    return code != 0 or GODOT_ERROR_RE.search(output) is not None


def reformat(args: argparse.Namespace) -> bool:
    check = [] if args.overwrite else ["--check"]
    ok = run([sys.executable, "-m", "gdtoolkit.formatter", *check, "scenes", "tests/godot", "dev"])[0] == 0
    ok &= run(["dotnet", "format", SOLUTION, *([] if args.overwrite else ["--verify-no-changes"])])[0] == 0
    ok &= run([sys.executable, "-m", "black", "--line-length", "100", *check, "dev"])[0] == 0
    return ok


def lint(args: argparse.Namespace) -> bool:
    return run([sys.executable, "-m", "gdtoolkit.linter", "scenes", "tests/godot", "dev"])[0] == 0


def lfs(args: argparse.Namespace) -> bool:
    # Fails if a file matching an LFS pattern was committed as a plain blob.
    return run(["git", "lfs", "fsck", "--pointers"])[0] == 0


BINARY_SUFFIXES = {".onnx", ".png", ".jpg", ".jpeg", ".webp", ".gif", ".ogg", ".wav", ".mp3", ".glb", ".blend",
                   ".ttf", ".otf", ".dll", ".so", ".dylib"}


def assets(args: argparse.Namespace) -> bool:
    """Every binary goes through LFS, and every painting has its prompt."""
    ok = True
    attributes = (REPO_ROOT / ".gitattributes").read_text(encoding="utf-8")
    tracked = set(run(["git", "ls-files"])[1].splitlines())
    for name in sorted(tracked):
        suffix = pathlib.Path(name).suffix.lower()
        if suffix in BINARY_SUFFIXES and f"*{suffix} " not in attributes and f"*{suffix}\t" not in attributes:
            print(f"{name}: add *{suffix} to the LFS patterns in .gitattributes", file=sys.stderr)
            ok = False
    prompts = REPO_ROOT / "dev" / "pictures" / "prompts"
    for painting in sorted((REPO_ROOT / "dev" / "pictures" / "painted").glob("*.png")):
        stem = painting.stem.removesuffix("_GOOD")
        if not (prompts / f"{stem}.md").exists():
            print(f"{painting.relative_to(REPO_ROOT)}: commit its prompt as dev/pictures/prompts/{stem}.md",
                  file=sys.stderr)
            ok = False
    return ok


def translations(args: argparse.Namespace) -> bool:
    ok = run([sys.executable, "dev/translations/extract.py", *([] if args.overwrite else ["--check"])])[0] == 0
    return ok and run([sys.executable, "dev/translations/check.py"])[0] == 0


def dotnet_test(args: argparse.Namespace) -> bool:
    return run(["dotnet", "test", SOLUTION])[0] == 0


def build(args: argparse.Namespace) -> bool:
    return run(["dotnet", "build", SOLUTION])[0] == 0


def godot_import(args: argparse.Namespace) -> bool:
    return not godot_failed(*run([args.godot, "--headless", "--path", ".", "--import"]))


def check_scripts(args: argparse.Namespace) -> bool:
    scripts = sorted(p for p in REPO_ROOT.rglob("*.gd") if not {"venv", "addons", ".godot"} & set(p.parts))

    def one(script: pathlib.Path) -> Tuple[pathlib.Path, bool, str]:
        code, output = run([args.godot, "--headless", "--path", ".", "--check-only", "--script", str(script)], 120)
        return script, not godot_failed(code, output), output

    return _parallel(one, scripts, args.jobs)


def godot_tests(args: argparse.Namespace) -> bool:
    tests = sorted((REPO_ROOT / "tests" / "godot").rglob("test_*.gd"))

    def one(test: pathlib.Path) -> Tuple[pathlib.Path, bool, str]:
        # With the fixed frame rate, Godot simulates the game time as fast as it can.
        code, output = run([args.godot, "--headless", "--fixed-fps", "60", "--path", ".", "--script", str(test),
                            "--", "--no-camera", "--mute", f"--settings=user://tests/{test.stem}.cfg"], 900)
        return test, "RESULT: OK" in output and not godot_failed(code, output), output

    return _parallel(one, tests, args.jobs)


def smoke_scenes(args: argparse.Namespace) -> bool:
    # The main scene and every test scene.
    scenes = ["res://scenes/main/main.tscn"] + [
        "res://" + p.relative_to(REPO_ROOT).as_posix() for p in sorted((REPO_ROOT / "scenes" / "tests").rglob("*.tscn"))]

    def one(scene: str) -> Tuple[str, bool, str]:
        code, output = run([args.godot, "--headless", "--path", ".", "--quit-after", "120", scene,
                            "--", "--no-camera", "--mute"], 300)
        return scene, not godot_failed(code, output), output

    return _parallel(one, scenes, args.jobs)


def _parallel(one: Callable, items: List, jobs: int) -> bool:
    ok = True
    with concurrent.futures.ThreadPoolExecutor(max_workers=jobs) as executor:
        for item, passed, output in executor.map(one, items):
            print(f"{'PASS' if passed else 'FAIL'} {item}", flush=True)
            if not passed:
                print(output)
                ok = False
    return ok


STEPS: Dict[str, Callable[[argparse.Namespace], bool]] = {
    "reformat": reformat, "lint": lint, "lfs": lfs, "assets": assets, "translations": translations, "dotnet-test": dotnet_test,
    "build": build, "import": godot_import, "check-scripts": check_scripts,
    "godot-test": godot_tests, "smoke-scenes": smoke_scenes,
}


# While prototyping: only the formatting, lint and typing, to iterate fast (section 1).
PROTOTYPE_STEPS = ["reformat", "lint", "build", "check-scripts"]


def main() -> int:
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("--overwrite", action="store_true", help="Fix the formatting and the translation files.")
    parser.add_argument("--prototype", action="store_true", help="Run only the formatting, lint and typing.")
    parser.add_argument("--select", nargs="+", choices=list(STEPS), default=None)
    parser.add_argument("--skip", nargs="+", choices=list(STEPS), default=[])
    parser.add_argument("--godot", default=os.environ.get("GODOT", "godot"))
    parser.add_argument("--jobs", type=int, default=os.cpu_count() or 2)
    args = parser.parse_args()
    selected = args.select or (PROTOTYPE_STEPS if args.prototype else list(STEPS))
    for name in [s for s in selected if s not in args.skip]:
        print(f"=== {name}", flush=True)
        if not STEPS[name](args):
            print(f"The step {name} failed.", file=sys.stderr)
            return 1
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

`gdformatrc` (gdtoolkit 4.5.0): `line_length: 100`, 4 spaces. gdlint's limit of 1000 lines per file forces healthy splits.

---

## 19. README, CONTRIBUTING, DESIGN, THIRD_PARTY_NOTICES

Four documents for four readers.
Change them **in the same commit** as the feature they describe, and keep each fact in one of them only.
*Why:* docs drifted in every game (a README still announcing "an experiment" at 1.0.0, a DESIGN claiming translations and fonts that did not exist, comments saying "wrists" where the code used hands).

### 19.1 README.md: for the players and their parents

```markdown
# <Game>

![<what the GIF shows, in words>](docs/gameplay.gif)

**<Game> is a free <genre> game for children, played by <moving how> in front of a webcam,
alone or with up to <n> friends.**

<What you do, in second person, 3–5 sentences. What it trains.>

**No network access.**
The game runs entirely on your computer and works offline.
The webcam images are processed locally, in memory, and are never saved or sent anywhere.

## Download

Get the latest version from the [releases](../../releases): Windows, macOS (Apple Silicon and Intel, macOS 11+), Linux.

| System | Webcam | Keyboard/mouse |
|---|---|---|
| Linux x86_64 | ✅ | ✅ |
| macOS 11+ | ✅ | ✅ |
| macOS 10.15 | ❌ | ✅ |
| Windows 10/11 | ✅ (no virtual cameras) | ✅ |

* **macOS:** the app is not notarized: open it the first time with a right click and *Open*.
* **Windows:** SmartScreen warns about an unknown app: *More info* → *Run anyway*. Allow the camera in *Settings → Privacy & security → Camera*.

## Setting up

![The room from above](docs/setup.svg)

* **Put the laptop on a chair or a low table, so that the camera is at the height of your hips (60–100 cm).** Not on the floor: …
* **Stand 2–3 m from the camera**, so that it sees you from the head to the waist.
* **Clear 2 × 2 m** around you.
* **Light the room from the front**, not from behind you.

## Acknowledgments

The models (<name, size, license, what for>), the paintings (made with an image generator from our prompts),
the sounds (thanks to <authors on Freesound>), the music (<credits>). See [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md); the design is in [DESIGN.md](DESIGN.md).

## License

<Game> is licensed under the [MIT License](LICENSE). The third-party components keep their own licenses.
```

### 19.2 CONTRIBUTING.md: the single manual for developers

1. **Overview**: the architecture in bullets (the vision in C#, the Godot-free core, the bridge, the game in GDScript, `dev/` never shipped).
2. **Setting up**: Git LFS, .NET 8, Godot 4.6.2 .NET, Python 3.10+, a webcam with 640x480, the tools of the asset scripts; one block to copy:
   ```
   export GODOT=~/apps/Godot_v4.6.2-stable_mono_linux_x86_64/Godot_v4.6.2-stable_mono_linux.x86_64
   git lfs install && git lfs pull
   python3 -m venv venv && touch venv/.gdignore
   venv/bin/pip install -r dev/requirements.txt
   dotnet build <Game>.sln
   $GODOT --headless --path . --import
   ```
   and in bold: **Do not skip the `.gdignore`!**
3. **Running the game**: `$GODOT --path . -- <flags>` and **every** flag: `--no-camera`, `--camera=<n>`, `--players=<n>`, `--lang=<code>`, `--mute`, `--no-voice`, `--skeleton`, `--record`, `--frames`, `--replay=<file>`, `--settings=<file>`, `--seed=<n>`, the developer keys.
4. **The tests**: the commands; what the end-to-end bot does and its contracts; how to replay a seed; where the logs of failed runs are (`user://tests/logs/`).
5. **Checks before a pull request**: `venv/bin/python dev/continuous_integration/precommit.py [--overwrite]` (`--prototype` while prototyping).
6. **Translations**: 11 in short: `tr()`, `# TRANSLATORS:`, the grammar, `extract.py`, `check.py`, adding a language, reviewing a draft.
7. **Pictures, sounds, music**: the pipelines and the license rules (14, 15).
8. **Recording**: consent, never commit, the tools.
9. **Packaging and releasing**: `dev/publish.py`, the version bump, the release.
10. **Coding rules**: classes not positional arrays; game time; the Godot-free core; `NOTE (mristin):` comments for the why; the units in the doc comments; the Godot gotchas (21).

### 19.3 DESIGN.md: for the designers

Status; the vision and how it serves the Manifesto; **what we dropped, and why**; the controls and the signals each level reads; calibration; sessions and saving; presentation; the vision pipeline with timings and its consequences for the design; safety and accessibility; open questions (struck through when decided).
Numbers from play tests go here with their date.

### 19.4 THIRD_PARTY_NOTICES.md: for the licenses

```markdown
# Third-party notices

<Game> is licensed under the MIT License; see LICENSE. This covers the code, the scenes, the shaders,
everything the game draws in code, and our transcriptions and renderings of the music.
The game includes, or is built with, the following third-party components. Each keeps its own license.
The packages carry all license texts under licenses/.

## Distributed with the game

### Godot Engine
* Version: 4.6.2
* License: MIT; its third-party components: licenses/godot/COPYRIGHT.txt
* Copyright: Juan Linietsky, Ariel Manzur and contributors
* Source: https://godotengine.org
* License text: licenses/godot/LICENSE.txt

### .NET runtime · ONNX Runtime 1.19.2 · CameraServerExtension (Windows only)
<the same bullets each>

### RTMO-t, RTMPose-t (MMPose) · YOLOX-Body-Head-Hand (PINTO model zoo)
* File: scenes/vision/rtmo/model/rtmo_t_body7_416x416.onnx (SHA-256 …)
* License: Apache-2.0 · License text: licenses/models/rtmo-LICENSE
* We distribute the models unmodified.
* The models were trained on the Body7 collection of datasets; check their terms before a commercial release.

### The paintings
Made with <generator> from our prompts (dev/pictures/prompts/). <The generator's terms on the rights in the output.>
**Altered:** cut from sheets, backgrounds removed, recoloured.

### The sounds
| Sound | Recording | Author | License |
|---|---|---|---|
| jump | [Boing 3](https://freesound.org/s/412345/) | someone | CC0 |

### The music
| Song | Work | Credit | License |
|---|---|---|---|
| lakes | [Title](https://commons.wikimedia.org/wiki/File:…mid) | … | CC0 |
Rendered with TiMidity and the FluidR3 GM SoundFont (MIT).

## Used only for development, not distributed
| Component | Version | License | Source |
|---|---|---|---|
| xUnit, CsCheck, StbImageSharp, gdtoolkit, black, NumPy, polib, … | | | |
```

`dev/write_godot_licenses.gd` writes Godot's license and its third-party notices from the engine; rerun it on every Godot upgrade:

```gdscript
#!/usr/bin/env -S godot --headless --script
extends SceneTree
## Write the license of Godot Engine and the notices of its third-party components
## into licenses/godot/, as the Godot binary reports them.
## Run: $GODOT --headless --script dev/write_godot_licenses.gd


func _initialize() -> void:
	var out := OS.get_environment("OUT_DIR")
	if out.is_empty():
		var script_path := ProjectSettings.globalize_path((get_script() as Script).resource_path)
		out = script_path.get_base_dir().get_base_dir().path_join("licenses").path_join("godot")
	DirAccess.make_dir_recursive_absolute(out)

	var license := FileAccess.open(out + "/LICENSE.txt", FileAccess.WRITE)
	license.store_string(Engine.get_license_text())
	license.close()

	var lines: PackedStringArray = []
	lines.append("Godot Engine includes the following third-party components.")
	lines.append("Generated from Engine.get_copyright_info() and Engine.get_license_info()")
	lines.append("of Godot " + Engine.get_version_info()["string"] + ".")
	lines.append("")
	for component in Engine.get_copyright_info():
		lines.append("Component: " + component["name"])
		for part in component["parts"]:
			for file in part["files"]:
				lines.append("  Files: " + file)
			for holder in part["copyright"]:
				lines.append("  Copyright: " + holder)
			lines.append("  License: " + part["license"])
		lines.append("")
	var licenses: Dictionary = Engine.get_license_info()
	var names := licenses.keys()
	names.sort()
	for name in names:
		lines.append("License: " + name)
		lines.append("")
		lines.append(licenses[name])
		lines.append("")
	var copyright := FileAccess.open(out + "/COPYRIGHT.txt", FileAccess.WRITE)
	copyright.store_string("\n".join(lines))
	copyright.close()
	quit(0)
```

The rest of `licenses/` is collected by `dev/publish.py` from its sources (the model folders, the NuGet cache, the addon), and the export fails if any is missing.

---

## 20. Game design for bodies

**Latency and recognition**
* From a cue to a recognized move takes **about 1 s**.
  Long telegraphs (1–1.4 s wind-ups), generous windows (from the second half of the wind-up to 0.5 s after), judge "as seen minus the latency", learn the latency per player (clamped to 0–0.6 s).
* Each level reads **only a few signals that do not compete** (running lifts the knees: no punches while running).
  Moving and steering are exclusive.
* Free play beats scripted rhythm; one new move per bar is too much for players and the vision.
* Children move differently from adults; tune thresholds on recordings of children (with consent).

**Space and setup**
* **Stay on the spot.**
  The room as a joystick (walking around) is slow, imprecise, confused with leaning, and loses the home spot.
* The camera at **hip height, 2.5–3 m away**; not on the floor (leaning looks like ducking).
  Light from the front.
* Turning means reaching **across the body's axis**, which avoids left/right confusion in the mirror.
* Check the clear space and tell the player to come closer or step back (6.7).

**Children, rest and the end of a session**
* Sessions with a clear end: levels of 3–5 minutes, a trip of about 10; save at the natural stops; nothing runs while the game is off.
* **Rest is part of the design**: calm screens between legs, gliding after strokes, things to look at and choose.
* Auto-pause when the player leaves the picture.
* **Never stuck**: after two failed tries the obstacle gives way; the game never needs precision a child cannot give.
* Forgiving and nonviolent: nothing explodes (it goes "poof"), defeated foes get dizzy and fly home, nobody dies, a late task gives fewer stars but never fails, the shore never hurts.
  Themes are friendly ("bad dreams", not "aliens"); no faces of real people.
* Rewards are earned by play (open a chest), not dropped at random.
* Tune with fine numbers (count damage in tenths), from play tests.
* Pictures for children who cannot read yet; stars, not grades; captions word by word at 0.4 s per word.
* Readable from 3 m: calm, dim backgrounds; big projectiles with bright outlines; status as thick rings on a dark backing.
* Warm-up and cool-down for every session.

**Together**
* Drop-in and drop-out; no names needed; shared rewards; rivals that adapt so the group never loses badly.
* Two players on one thing (a boat, a crane): unsynchronized moves wobble, synchronized ones get a bonus, which invites counting together.
* Cooperation is never slower than playing alone; the bots check it.

**Accessibility**
* **Read aloud** with `DisplayServer.tts_*` (offline; Linux needs `speech-dispatcher` and `espeak-ng`): off by default, toggled with V and remembered; rate 0.9; never the same text twice within 2 s; the voice per language from the `voice` of `languages.json`; a button speaks when a hand comes onto it, a screen speaks its heading.
* Colour-blind-safe player colours (shape and colour), a seated variant where the game allows it, sound cues for visual cues.
* Keyboard and mouse always work.

---

## 21. Godot gotchas

* **Godot exits with 0 on many errors**: grep the output of every headless run for `ERROR`, `SCRIPT ERROR`, `Parse Error`.
* **Import before checking**: `--import` refreshes the cache of the global class names that `--check-only` and the tests rely on.
* **`Geometry2D.is_point_in_polygon` is single precision** and double-counts a vertex the ray passes through: it put invisible walls into the sea.
  Write your own crossing count (ray east, half-open edges, each polygon separately):
  ```gdscript
  static func inside(point: Vector2, polygon: PackedVector2Array) -> bool:
  	var result := false
  	var j := polygon.size() - 1
  	for i in polygon.size():
  		var a := polygon[i]
  		var b := polygon[j]
  		if (a.y > point.y) != (b.y > point.y):
  			if point.x < a.x + (point.y - a.y) * (b.x - a.x) / (b.y - a.y):
  				result = not result
  		j = i
  	return result
  ```
* **Physics push-outs count against the speed limit**: a follower that walked its full step and was then pushed out of a wall moved too far; count the push against the step, and slide along the edge.
* **Hold on to the textures you draw**: a texture that nothing references is freed before the frame shows and draws white.
* **Script translations** must be removed with `TranslationServer.remove_translation` in `_exit_tree`, or the engine crashes when it frees them.
* `set_deferred("monitoring", …)` inside physics callbacks; add projectiles deferred, after setting their position, so they know where they start in `_ready`.
* A `Timer` node, not `get_tree().create_timer`, for anything whose outcome must not fire after the level is gone.
* Lazy `load()` instead of `preload` to break cyclic scene references.
* Cooldowns: `cooldown = maxf(cooldown + interval, 0.0)`: do not catch up on shots missed during a long frame.
* Fake controllers emit their first signal with `call_deferred`: the game connects only after adding them.
* **Heavy backgrounds**: render a shader once into a `SubViewport` tile (`UPDATE_ONCE`; vertically periodic) and only slide the texture; weak laptops cannot draw it every frame.
* Deterministic procedural art: seed with `hash(name) ^ hash(Vector2i(position))`, so it looks the same in the editor and the game.
* The `.po` files and the JSON data are not resources: add them to `include_filter`, or the export leaves them out.
* MSBuild reads environment variables as properties: never call a CI variable `VERSION`.

---

## 22. Checklist for a new game

1. Copy the layout of section 2; `.gitattributes`, `.gitignore`, every `.gdignore`; `git lfs install`.
2. `project.godot`, the `.csproj`, the export presets (sections 2, 5.6, 16).
3. Copy the vision (`scenes/vision/`), its tests and the camera extension; check that the models load and are not LFS pointers.
4. `precommit.py` and `ci.yaml` with `--prototype`: formatting, lint and typing only.
5. Prototype the game until it is playable end to end; every text a whole sentence in `tr()`.
6. A play test with children (with consent) before tuning anything.
7. Once the game works: the unit and property-based tests, the fake vision, the end-to-end bot, the contracts and the error log.
8. The translations: `languages.json` with English and at least one more language; `Languages`, `Grammar`, extraction and checks.
9. Drop `--prototype`; `publish.py`, the publish workflow; a first dry-run publish.
10. README, CONTRIBUTING, DESIGN, THIRD_PARTY_NOTICES from section 19; `dev/write_godot_licenses.gd`.
11. The style brief and the first sketch, prompt and painting; the sound and music manifests.
