# Understanding the Xbox Dashboard

How the retail Xbox dashboard works, from power-on to green orb. This is a technical walkthrough of the application architecture based on reverse engineering work, recovery disc analysis, and XAP script examination.

## Power On to Dashboard

When you turn on an Xbox, the kernel boots from flash ROM, initializes the hardware, mounts the hard drive partitions using internal NT device paths (`\Device\Harddisk0\Partition1` through `\Partition6`), and launches the dashboard executable: `xboxdash.xbe` from partition 2.

The dashboard is a single-threaded Direct3D 8 application. It's not a shell, not a window manager, not an OS component. It's a regular Xbox executable that happens to be the first thing the kernel runs. It has the same access to hardware that any game does, plus some extra kernel privileges for system management (formatting memory units, setting clock/language, launching other XBEs).

## The Application

The entire dashboard is driven by a singleton application object. On startup, it runs through an initialization sequence:

1. **Memory system** -- Sets up the application's memory pools
2. **Drive mapping** -- Creates symbolic links from drive letters to kernel partition paths. The retail dashboard maps three: `C:` (system), `Y:` (dashboard data), `T:` (title data)
3. **Disc drive** -- Initializes the DVD/CD drive for auto-detection of inserted media
4. **Direct3D 8** -- Creates the D3D device, sets up the back buffer (480p or widescreen), initializes the projection matrix
5. **Audio** -- Initializes DirectSound for ambient audio, sound effects, and music playback
6. **XIP loading** -- Loads `default.xip` from the dashboard data directory. This archive contains all the root scripts and shared textures
7. **Script class system** -- Initializes the node class registry so the script parser knows what node types exist
8. **Title enumeration** -- Scans the hard drive for installed game titles and save data
9. **Material system** -- Initializes the skin/material lookup system
10. **Script loading** -- Loads and compiles `default.xap`, the root script file. This creates the scene graph
11. **Scene instantiation** -- Creates the root node from the compiled class, building the full 3D scene
12. **Script initialization** -- Calls the `initialize()` function in the root script, which sets up the main menu, selects the default menu item, and starts the ambient audio

After initialization, the main loop is two function calls, forever:

```
for (;;)
{
    Advance();  // update time, input, animations, scripts
    Draw();     // render the scene
}
```

No event queue. No message pump. Just tick and render, every frame, until the console is turned off or another XBE is launched.

## The Scene Graph

Everything visible on screen is a node in a tree. The root node is an instance of the class defined by `default.xap`. That instance contains child nodes for the main menu, the memory manager, the music player, the settings screens -- each loaded as an `Inline` node that references another XAP file inside another XIP archive.

### Node Types

Every node is a C++ class registered with the script engine through a reflection system. The key node types:

| Node | What It Does |
|------|-------------|
| **Screen** | Display resolution, brightness, framerate |
| **Background** | Sky color and backdrop texture |
| **NavigationInfo** | Camera near/far planes, visibility range |
| **Viewpoint** | Camera position, orientation, field of view |
| **Group** | Container for child nodes |
| **Transform** | Position, rotation, scale applied to children |
| **Level** | A scene with its own viewpoint and input handler (menu sections) |
| **Inline** | Loads another XAP/XIP as a child scene |
| **Layer** | Overlay scene rendered on top (keyboard, message dialogs) |
| **Shape** | Visible geometry with an appearance and mesh |
| **Mesh** | Vertex/index buffer loaded from an .xm file |
| **Sphere** | Procedurally generated sphere mesh |
| **Text** | 3D text rendered from Xbox fonts |
| **ImageTexture** | Texture loaded from a .xbx file |
| **MaxMaterial** | Material with named properties that the skin system can override |
| **AudioClip** | Sound effect or looping ambient audio |
| **PeriodicAudioGroup** | Randomly triggered ambient sounds (steam hisses, alien voices) |
| **TimeSensor** | Time-based activation/deactivation |
| **Joystick** | Controller input with button/stick callbacks |
| **Config** | System settings (language, video mode, parental controls) |
| **SavedGameGrid** | Title/save game enumeration with icon rendering |
| **MusicCollection** | Soundtrack enumeration and playback |
| **DVDDrive** | Disc type detection (game, video, audio) |
| **Translator** | Localization string lookup |
| **Settings** | INI-format .xbx file reader/writer |
| **Folder** | File system directory enumeration |
| **Keyboard** | Virtual keyboard for text input |

### The Reflection System

Each node class registers its properties and functions through macro tables that embed metadata as string literals in the binary:

```cpp
IMPLEMENT_NODE("Joystick", CJoystick, CNode)

START_NODE_PROPS(CJoystick, CNode)
    NODE_PROP(pt_boolean, CJoystick, isBound)
    NODE_PROP(pt_number,  CJoystick, frequency)
    NODE_PROP(pt_number,  CJoystick, xaxis)
    NODE_PROP(pt_number,  CJoystick, yaxis)
END_NODE_PROPS()

START_NODE_FUN(CJoystick, CNode)
    NODE_FUN_VV(OnMoveUp)
    NODE_FUN_VV(OnMoveDown)
    NODE_FUN_VV(OnADown)
    NODE_FUN_VV(OnBDown)
END_NODE_FUN()
```

`PRD` (Property Descriptor) stores the offset into the C++ object, the type, and the name as a string. `FND` (Function Descriptor) stores a member function pointer, its signature type, and the name. When a script says `myJoystick.xaxis`, the engine looks up `"xaxis"` in the Joystick property table, finds the byte offset, and reads the float directly from the C++ object's memory.

This is what made reverse engineering tractable: every property name, every function name, every class name is embedded as a string in the binary. Find one node's tables, and the pattern repeats for all of them.

## The Script Engine

### Compilation

XAP scripts are plain text compiled to bytecode at load time. The pipeline:

1. **Lexer** -- Tokenizes the text into keywords, identifiers, numbers, strings, operators
2. **Parser** -- Parses VRML97 node declarations and JavaScript-like function bodies into an AST
3. **Compiler** -- Generates stack-based bytecode from the AST
4. **Function storage** -- Compiled bytecode is stored as `CFunction` objects attached to nodes

The bytecode uses a stack machine design. Operations push and pop values from a 256-element object stack. A typical property assignment like `myPanel.visible = true` compiles to:

```
opThis          // push 'this'
opDot "myPanel" // resolve child node
opNum 1.0       // push true
opAssign "visible" // set property
```

### Runtime

The script VM (CRunner) executes bytecode one instruction at a time. It supports:

- **Variables**: local, instance, and global scope
- **Control flow**: if/else, for, while, break, return
- **Expressions**: arithmetic, comparison, logical, string concatenation
- **Function calls**: both script functions and native C++ methods
- **Sleep**: `sleep(seconds)` pauses execution and resumes next frame
- **Eval**: `eval(string)` compiles and executes a string as code at runtime

Each node can have a `behavior` -- a CRunner instance that executes attached script functions. When a Joystick node detects a button press, it calls `CallFunction(node, "OnADown")`, which looks up the function in the node's script, creates a runner, and executes the bytecode.

### Built-in Objects

The VM provides JavaScript-like built-in objects:

- **Math** -- sin, cos, sqrt, pow, random, PI, etc.
- **String methods** -- length, charAt, indexOf, substring, substr, split, toUpperCase, toLowerCase
- **Type conversion** -- automatic number/string coercion
- **new** -- `new Settings`, `new File`, `new Folder` create native objects from script

## Rendering

### Per-Frame Draw

Each frame:

1. **Projection** -- Compute the perspective matrix from the active Viewpoint's field of view and NavigationInfo's near/far planes
2. **Clear** -- Clear the back buffer to the Background's sky color
3. **Backdrop** -- If the Background has a backdrop texture, render it as a full-screen quad
4. **Scene traversal** -- Walk the node tree recursively. Each Group pushes its transform onto the matrix stack, renders its children, then pops. Each Shape sets its material and draws its mesh.
5. **Present** -- Swap the back buffer to the screen

### Meshes

Mesh data comes in two forms:

**Loose .xm files** (alpha builds): A `MESHFILEHEADER` followed by raw vertex data and index data. The header specifies primitive type, vertex count, index count, FVF (Flexible Vertex Format), and vertex stride.

**XIP-packed meshes** (retail): Multiple meshes share index buffers and vertex buffers inside the archive. Each named mesh is a `mesh reference` that points to a range within a shared buffer -- buffer index, first index offset, and primitive count. This reduces draw call overhead by minimizing buffer switches.

The FVF flags describe what each vertex contains:
- Position (xyz)
- Normal (packed 3-component)
- Diffuse color (ARGB)
- Texture coordinates (uv)

Different combinations use different vertex shaders. The `falloff` shader system provides five vertex programs (effect through effect4, plus aniso) that compute viewing-angle transparency -- the normal dot position calculation that gives the dashboard its X-ray glass look.

### Materials and Textures

`MaxMaterial` nodes connect script-side material names to textures. When the renderer encounters a Shape with a MaxMaterial, it looks up the material name in the skin system:

1. Check the current skin folder for a matching .xbx file
2. Check loaded XIPs for a matching texture
3. Fall back to the default texture

This lookup order is what makes skins work. A skin folder with `cellwall.xbx` overrides the cellwall texture from the XIP without modifying the archive. The skin system is just the texture search path.

### The Falloff Shader

The signature Xbox dashboard look -- translucent green pods that glow at the edges -- comes from a custom NV2A vertex shader. The shader source (`effect3.vsh`, found on a 3729 recovery disc):

```
// compute fall-off effect colors; D = normal dot position
dp3 r0.w, r0, r2
mov r1, c16
max r0.w, r0.w, -r0.w
mad oD0, r1, r0.w, c15
```

It computes the dot product between the surface normal and the view direction. Surfaces facing the camera are transparent (front color). Surfaces at glancing angles are opaque (side color). The absolute value ensures both sides of the mesh get the effect. The result is a lerp between two colors based on viewing angle.

The C++ side (`SetFalloffShaderValues`) passes the front color, side color, and transformation matrices as vertex shader constants. Each mesh can specify whether it uses the falloff effect, and with what colors.

## Audio

The dashboard's ambient audio system creates the submarine/space station atmosphere:

**Looping soundscapes** -- Three ambient tracks (HYDROTHUNDER, COMMUNICATION, ENGINEROOM) play continuously at low volume. Each menu section crossfades to a different track.

**Periodic random events** -- `PeriodicAudioGroup` nodes trigger random sounds at configurable intervals:
- Steam hisses: every 60-80 seconds, 7 variants
- Alien voices: every 120-150 seconds, 13 variants (modified Apollo mission recordings)
- Comm chatter: every 60-95 seconds, 9 voice clips + 4 static bursts, stereo panned left/right
- Pinger: every 300-360 seconds, rare deep-space ping

**Sound effects** -- Button presses, menu transitions, error beeps, progress bars. Each has a named AudioClip node that scripts trigger with `.Play()`.

## Input

The `Joystick` node polls controller state at a configurable frequency and fires callbacks:

- **D-pad / stick**: `OnMoveUp`, `OnMoveDown`, `OnMoveLeft`, `OnMoveRight`
- **Buttons**: `OnADown`/`OnAUp`, `OnBDown`/`OnBUp`, `OnXDown`, `OnYDown`
- **Triggers**: `OnLeftDown`, `OnRightDown` (analog triggers as digital buttons)
- **Shoulder**: `OnBlackDown`, `OnWhiteDown`
- **System**: `OnStartDown`, `OnBackDown`

The Joystick node handles typomatic repeat (initial delay, then repeat interval) for stick/d-pad input, dead zone processing, and screen saver reset on any input. Only one Joystick node is "bound" at a time -- menu transitions rebind input to the new section's controller handler.

## The Menu System

The main menu uses `Level` nodes -- self-contained scenes with their own viewpoints, input handlers, and tunnel transitions. When you press A on "Music," the script calls `GoToMusic()`, which makes the Music Level's viewpoint active and binds its Joystick handler.

```javascript
DEF theMusicInline Inline
{
    visible false
    url "Music.xap"

    function onLoad()
    {
        theMusicInline.children[0].theMusicMenu.GoTo();
    }
}

function GoToMusic()
{
    if (theMusicInline.visible)
        theMusicInline.children[0].theMusicMenu.GoTo();
    else
        theMusicInline.visible = true;  // triggers load + onLoad
}
```

Setting `visible = true` on an Inline triggers the XIP load. The `onLoad` callback fires when the archive is loaded, and `GoTo()` activates the Level's viewpoint. The camera smoothly interpolates to the new position (the `jump false` property on viewpoints enables smooth transitions).

Pressing B calls `GoBackTo()`, which returns to the previous Level's viewpoint. The menu system is a stack of viewpoints with smooth camera transitions between them.

## The Time Bug

The xapp header contains this comment:

```cpp
// Represent time values as a 'double' for good fractional precision after
// a large period of time (days).  The really proper fix would be to represent
// time using a DWORD, and convert to floating point after computing an
// age, but we don't have time to do that...
typedef double XTIME;
```

They didn't have time to fix the time code. Twenty-five years later, nobody else has either.

## Animations: The Lerper

Every smooth transition in the dashboard -- menu items fading in, pods rotating, cameras moving between viewpoints -- is driven by the Lerper system. A `CLerper` is a dead-simple linear interpolation over time:

```cpp
bool CLerper::Advance()
{
    float t = (XAppGetNow() - m_startTime) / m_interval;
    if (t >= 1.0f)
    {
        *m_pValue = m_endValue;
        return false; // done, remove me
    }
    *m_pValue = (1.0f - t) * m_startValue + t * m_endValue;
    return true; // still running
}
```

Set a target value and a duration, and the Lerper writes the interpolated value directly to the target float every frame. When it's done, it deletes itself from the linked list. `CLerper::AdvanceAll()` ticks every active Lerper in the system, once per frame, before the scene graph advances.

When a script calls `node.SetTranslation(x, y, z)` or `node.SetRotation(ax, ay, az, angle)`, those functions create Lerpers that smoothly animate the node to its new position over a short interval. The `fade` property on nodes creates a Lerper that animates opacity. This is why menu transitions feel smooth -- nothing snaps, everything glides.

## The `fade` Property

Deselected menu items in the main menu have `fade 0.25` -- they're dimmed to 25% opacity. When the cursor moves, the previous item gets a new Lerper targeting 0.25, and the newly selected item gets one targeting 1.0. The crossfade happens over a few frames, giving the menu its characteristic glow-and-dim selection feel.

The `fade` property is different from the `falloff` shader. Fade is simple alpha multiplication applied uniformly to a node and all its children. Falloff is a per-pixel viewing-angle effect computed in the vertex shader. Fade makes things transparent uniformly; falloff makes them transparent based on surface angle.

## Text Rendering

Text in the dashboard is rendered as 3D geometry. Each character is a tessellated glyph shape -- vertices and indices describing the outline of the letter, filled as triangles. The font data comes from `.xtf` files (Xbox Text Font), which contain:

- Glyph shapes: vertex positions and triangle indices for each character
- Glyph metrics: black box size, origin, cell advance (spacing)
- Unicode range tables: which characters are available in the font

When a `Text` node renders, it walks the string character by character, looks up each glyph's shape, positions it using the advance metrics, and draws the triangles with the current material color. Text is real 3D geometry in the scene -- it has depth, responds to lighting, and can be transformed like any other mesh.

The dashboard ships with two fonts: `Xbox.xtf` (the main UI font) and `Xbox Book.xtf` (the serif font used for body text). The `font` property on Text nodes selects between them: `font "Heading"` for the main font, `font "Body"` for the book font.

A nasty hack lives in the text renderer: `g_nTextChar` is a global index for placing a cursor in the virtual keyboard's text field. The comment says "Oh what a nasty hack..." -- and it's been there since the original.

## The Config Node: EEPROM Access

The `Config` node exposes the Xbox EEPROM settings to scripts. The EEPROM is a small persistent storage on the motherboard that holds:

- Language (1-6: English, Japanese, German, French, Spanish, Italian)
- Video mode (normal, letterbox, widescreen)
- Video resolution support (480p, 720p, 1080i, PAL-60)
- Audio mode (mono, stereo, Dolby Surround)
- Dolby Digital and DTS toggle
- Parental control password and ratings (separate for games and movies)
- Time zone and daylight saving time
- Auto power-off setting

Config reads and writes directly to the EEPROM through kernel calls. When the settings script changes the language, `Config.SetLanguage()` writes to the EEPROM and the change persists across reboots. The `GetLaunchReason()` method checks why the dashboard was launched -- cold boot, return from a game, memory management request, or error recovery.

The Config node also provides system info methods: `GetAVPackType()` (what AV cable is connected), `GetAVRegion()` (NTSC/PAL), `GetGameRegion()`, `GetROMVersion()`, `GetXdashVersion()`, and `GetRecoveryKey()` (the Xbox support recovery key).

## Localization: The Translator

The `Translator` node handles all localization. Translation data is stored as key-value pairs in language-specific sections of the XBE:

```
EnglishXlate section:
MEMORY="MEMORY"
MUSIC="MUSIC"
SETTINGS="SETTINGS"
BACK="BACK"
SELECT="SELECT"
```

Up to 500 translation terms per language (the `MAX_XLATE` limit). When a script calls `theTranslator.Translate("MEMORY")`, the Translator looks up the key in the current language's table and returns the localized string. Six languages are supported: English, Japanese, German, French, Spanish, Italian.

The language files in the alpha builds were plain `.txt` files with escape codes (`\xA9` for copyright, `\u2122` for trademark). By retail, the data was compiled into XBE sections for faster loading.

## Level Transitions: Tunnels and Cameras

When you navigate between menu sections, the `Level` node orchestrates the transition:

1. **GoTo()** is called on the target Level
2. The target Level's `tunnel` mesh becomes visible (the 3D tube connecting menu sections)
3. The camera binds to the Level's `path` -- either a Viewpoint (instant jump) or a CameraPath (animated fly-through)
4. A timer starts (`g_timeToNextLevel`) for the transition duration
5. `OnActivate` fires on the target Level's script
6. While the timer runs, the tunnel mesh renders and the camera moves
7. When the timer expires, the tunnel hides, the previous Level deactivates, and the Joystick rebinds to the new Level's `control` node
8. `OnArrival` fires -- the destination Level is now active

Going back (`GoBackTo`) reverses the process -- the camera follows the *departing* Level's path in reverse, creating the feeling of backing out through the tunnel you came in through.

The `archive` property on Level nodes specifies which XIP to load for that section. When the Memory Level loads `Memory2.xip`, the Level node triggers the XIP load and waits for it to finish before firing `OnArrival`. This is how the dashboard lazy-loads sub-scenes -- XIPs are only read from disk when you navigate to them.

The `unloadable` flag allows the cache system to evict a Level's XIP meshes when memory gets tight. `CleanupMeshCache()` scans for the oldest unlocked XIP with mesh buffers and releases them. When you return to that Level, the meshes reload from disk.

## Screen Saver

The ScreenSaver node watches for idle time and fires callbacks:

- **Default delay**: 300 seconds (5 minutes) of no input triggers `OnStart`, which dims the screen brightness
- **delay2 / delay3**: Additional thresholds for multi-stage screen savers (the alpha builds had plans for more elaborate idle animations)
- **reset()**: Any controller input resets the timer. The Joystick node calls `ResetScreenSaver()` on any button press or stick movement

The retail screen saver just dims brightness: `theScreen.brightness = 0.1` in `OnStart`, `theScreen.brightness = 1.0` in `OnEnd`. On Xbox hardware, `ResetScreenSaver()` also calls `XAutoPowerDownResetTimer()` to prevent the console from auto-shutting off.

## The DVD Player

The DVD player is its own subsystem with a separate XIP (`dvd.xip`). When the disc drive detects a video DVD:

1. `DVDDrive.OnDiscInserted()` fires with `discType == "Video"`
2. The script calls `StartDVDPlayer()`, which makes the DVD player Inline visible
3. The DVD Inline loads `dvd.xap`, which creates the playback UI
4. The Ravisent DVD decoder library handles actual video decoding

The DVD player has its own viewpoint, Joystick handler, and full transport controls. The `dvdstop.xbx` texture (and its widescreen variant `dvdstopw.xbx`) is the "Insert a DVD" splash screen.

Audio CDs follow a similar path: `discType == "Audio"` triggers `StartCDPlayer()`, which redirects to the Music section with CD playback mode.

## What Gets Called Every Frame

To summarize the per-frame execution:

**Advance()** (update):
1. Compute delta time from `GetTickCount()`
2. Advance the camera interpolation
3. Tick all active Lerpers (animation system)
4. Walk the scene graph recursively -- each node's `Advance()`:
   - CGroup advances all children
   - CNode runs its behavior script (CRunner) if one is attached
   - TimeSensor checks activation windows
   - AudioClip handles playback state
   - Joystick polls controller input and fires callbacks
   - ScreenSaver checks idle timers
   - Level checks transition timers

**Draw()** (render):
1. Update projection matrix if the viewpoint changed
2. Clear to Background sky color
3. Render backdrop texture if present
4. Push identity world matrix
5. Walk the scene graph recursively -- each node's `Render()`:
   - Transform multiplies its matrix onto the stack
   - Shape sets material/texture, draws mesh
   - Text tessellates glyphs and draws triangles
   - Sphere generates and draws a procedural sphere
   - Level renders its tunnel mesh if visible
   - Group renders all visible children
6. Pop world matrix
7. Present the back buffer

Two function calls. Everything else is recursion.
