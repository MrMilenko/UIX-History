# XAP Script System

## Overview

XAP is the scripting language that defines the entire Xbox dashboard UI. Every panel, button, text label, 3D mesh, animation, and interaction in the dashboard is defined in XAP scripts -- the C++ engine just runs them. This is what made the Xbox dashboard moddable: you didn't need to touch the XBE to reshape the entire user experience.

The language is a hybrid of **VRML97** (for declaring 3D scene nodes) and **JavaScript** (for scripting logic). This wasn't a coincidence or an "inspired by" design choice. Microsoft's build pipeline included `wrl2xm`, a tool that literally converted VRML `.wrl` files from 3D modeling tools into Xbox mesh format. The scene description language IS VRML97, extended with JS-like scripting blocks for interactivity.

`.xap` files are plain text. The dashboard's built-in compiler converts them to bytecode at load time -- not pre-compiled on disk. This meant anyone with a text editor could modify the dashboard UI, test on hardware, and iterate. No compiler toolchain needed. This is why the XAP modding scene exploded in the early 2000s.

## What a XAP File Looks Like

Here's a simplified example of a dashboard panel:

```vrml
DEF myPanel Panel {
    position 0 0 0
    scale 1 1 1
    opacity 1.0

    children [
        DEF title Text {
            string "Welcome"
            fontStyle FontStyle {
                family "Xbox"
                size 24
            }
        }

        DEF icon ImageTexture {
            url "icon.xbx"
        }

        Shape {
            geometry Mesh { url "panel_bg.xm" }
            appearance Appearance {
                material Material {
                    diffuseColor 0.2 0.8 0.2
                    transparency 0.1
                }
            }
        }
    ]

    Script {
        function onActivate() {
            title.string = "Selected!";
            this.opacity = Lerp(this.opacity, 1.0, 0.1);
        }

        function onDeactivate() {
            title.string = "Welcome";
            this.opacity = Lerp(this.opacity, 0.5, 0.1);
        }
    }
}
```

This creates a panel containing a text label, a texture, and a 3D mesh background. Script functions respond to navigation events and animate properties.

## The Compilation Pipeline

```
.xap text file
    |
    v
[Lexer] -- Tokenizes into: identifiers, numbers, strings, operators, keywords
    |
    v
[Parser] -- Builds AST: node declarations, property assignments, script blocks
    |        Resolves DEF/USE references (VRML97 naming)
    |        Handles Inline nodes (lazy sub-scene loading)
    |
    v
[Compiler] -- Generates bytecode for script blocks
    |          Resolves property/function references against node reflection tables
    |          Produces compact instruction stream
    |
    v
[Runner VM] -- Executes bytecode at runtime
               Stack-based, 100-element stack
               Operations: push, pop, call, get/set property, arithmetic, string ops
```

### The Lexer

The lexer (`Lexer.cpp`) is a hand-written scanner that recognizes:
- **Keywords**: `DEF`, `USE`, `Script`, `function`, `if`, `else`, `while`, `for`, `return`, `var`, `true`, `false`, `null`
- **Identifiers**: Node names, property names, function names
- **Numbers**: Integer and floating-point literals
- **Strings**: Double-quoted string literals with escape sequences
- **Operators**: `+`, `-`, `*`, `/`, `==`, `!=`, `<`, `>`, `<=`, `>=`, `&&`, `||`, `!`, `=`, `.`
- **VRML tokens**: `{`, `}`, `[`, `]`, commas (optional in value lists)

### The Parser

The parser (`Parser.cpp`) handles two distinct grammars:

1. **VRML97 scene declarations**: Node types, property values, child lists, `DEF`/`USE` names
2. **JavaScript-like script blocks**: Everything inside `Script { ... }` blocks

When the parser encounters a node type name (like `Panel`, `Text`, `Shape`), it looks it up in the global node registry, the same registry populated by `IMPLEMENT_NODE` macros. If the type exists, it creates an instance and begins parsing its properties and children.

### The Compiler

The compiler (`Compiler.cpp`) converts script ASTs into bytecode. It handles:
- Variable declarations and scoping
- Property access chains (`myPanel.children[0].opacity`)
- Function calls (both script-defined and native node functions)
- Arithmetic and comparison operations
- Control flow (`if`/`else`, `while`, `for`)
- String concatenation and comparison

### The Runner VM

The runner (`Runner.cpp`, `Runner.h`) is a stack-based virtual machine:

- **Stack**: Fixed array of 100 elements. Each element can hold a number, string, node reference, or array.
- **Execution**: The VM reads bytecode instructions sequentially, pushing/popping the stack.
- **Property access**: When a script reads `myPanel.opacity`, the VM looks up the property in the node's reflection table and calls the registered getter function.
- **Function calls**: When a script calls `myPanel.SetPosition(x, y, z)`, the VM pushes arguments, looks up the function in the reflection table, and calls the registered C++ function.

Key VM types defined in `Runner.h`:
- `CRunner`: The VM execution context
- `CNumObject`: Numeric value (double precision)
- `CStrObject`: String value (reference counted)
- `CMember`: Variant type that can hold numbers, strings, nodes, or arrays
- `CInstance`: Per-node script state (local variables, function pointers)

## The Node Reflection System

Every node class in the dashboard uses a macro-based reflection system that lets the script VM discover properties and functions at runtime.

### Declaring a Node

```cpp
// In the .h file:
class CPanel : public CGroup {
public:
    CPanel();
    DECLARE_NODE(CPanel, CGroup)

    float m_opacity;
    void SetOpacity(float val);
    float GetOpacity();
};
```

### Implementing a Node

```cpp
// In the .cpp file:
IMPLEMENT_NODE("Panel", CPanel, CGroup)

START_NODE_PROPS(CPanel, CGroup)
    NODE_PROP_GS(pt_number, "opacity", GetOpacity, SetOpacity)
    NODE_PROP_GS(pt_string, "name", GetName, SetName)
END_NODE_PROPS()

START_NODE_FUN(CPanel, CGroup)
    NODE_FUN_IV("Show", Show)       // void Show(int)
    NODE_FUN_NV("SetScale", SetScale) // void SetScale(float)
END_NODE_FUN()
```

The macros expand to static arrays of property and function descriptors. When the parser encounters `Panel { opacity 0.5 }`, it:

1. Looks up "Panel" in the node type registry
2. Creates a `CPanel` instance
3. Looks up "opacity" in `CPanel`'s property table
4. Calls the registered setter with the parsed value

This is how the same scripts work on both Xbox and desktop. The node classes register the same properties and functions. On desktop, some nodes are stubs (all getters return defaults, all setters are no-ops), but the script VM doesn't care.

## Script Features

### Variables and Types
```javascript
var count = 0;
var name = "Player One";
var panel = myPanel;        // Node reference
var items = new Array();    // Dynamic array
```

### Property Access
```javascript
myPanel.opacity = 0.5;
var pos = myPanel.position;
theConfig.SetValue("skin", "Carbon");
```

### Control Flow
```javascript
if (count > 10) {
    label.string = "Too many!";
} else {
    count = count + 1;
}

while (i < items.length) {
    items[i].opacity = 0;
    i = i + 1;
}

for (var i = 0; i < 10; i = i + 1) {
    DoSomething(i);
}
```

### Built-in Functions
```javascript
Lerp(a, b, t)              // Linear interpolation
Translate("MENU_TITLE")    // Localized string lookup
ToString(42)               // Number to string
ParseInt("42")             // String to number
```

### Events
Nodes emit events that scripts can handle:
```javascript
function onActivate() { }      // This node gained focus
function onDeactivate() { }    // This node lost focus
function Advance(time) { }     // Called every frame
function onClick() { }         // A button pressed
```

## Inline Nodes: Lazy Loading

The dashboard doesn't load everything at once. The root `default.xap` references sub-scenes via `Inline` nodes:

```vrml
DEF gamesPage Inline {
    url "games.xap"
}
```

When the user navigates to the games page, the `Inline` node triggers loading of `games.xap`, which is parsed, compiled, and added to the scene graph on the fly. This keeps startup fast and memory usage low.

## The .xbx File Format Decision

The exploits documented below weren't figured out by one person or one team. They accumulated over years across the Xbox modding scene -- tHc, BSX, UIX, and dozens of people experimenting with scripts on real hardware, sharing what worked on forums and in IRC. By the time UIX Lite pulled them together into a cohesive dashboard, the individual tricks had been floating around the community for years. The XAP modding scene was collaborative in the way that early-2000s console hacking scenes tended to be: someone would post a discovery, someone else would build on it, and eventually it became common knowledge that nobody could trace back to a single origin.

Before diving into the specifics, it's worth understanding the single decision that made all of this possible.

Some engineer at Microsoft decided to use the `.xbx` extension for settings files. Not `.ini`, not `.txt`, not `.cfg` -- `.xbx`, the same extension as the binary texture format. `TitleMeta.xbx` was a plain UTF-16LE text file sitting in UDATA next to actual DXT-compressed textures. And unlike XIPs (signed) and XBEs (signed), these `.xbx` text files were **unsigned**. You could write whatever you wanted.

This meant the `Settings` object -- which Microsoft exposed to scripts for reading these metadata files -- could read and write arbitrary INI-format text to any `.xbx` file on the hard drive. And since `.xbx` files weren't signed, those changes persisted. The community realized that if you could read and write arbitrary key-value pairs to disk from script, you could build configuration systems, title caches, icon databases, quick launch bindings, Discord presence state, and skin metadata -- all stored in files that the console trusted implicitly because they had the same extension as textures.

Every hack that follows traces back to this. The `Settings` object was Microsoft's read-only metadata viewer. The unsigned `.xbx` extension turned it into a general-purpose database.

### The Ghost of games.xap

Here's the thing nobody realized until we dug through the alpha builds: the `.xbx` text file convention wasn't invented for the memory manager. It was invented for `games.xap` -- the game launcher that Microsoft built in early 2001 and killed before retail.

The alpha game launcher used `Info.xbx` for per-game news feeds, `Menu.xbx` for custom menu definitions, `TitleMeta.xbx` for publisher metadata, and `TitleImage.xbx` for cover art. All plain text, all writable, all stored in per-game directories on the hard drive. The entire `.xbx` text file convention was designed so game publishers could ship dashboard-readable metadata alongside their titles.

When Microsoft cut the game launcher in the summer of 2001, the `.xbx` convention survived in a reduced form. `TitleMeta.xbx` and `SaveMeta.xbx` stayed because the memory manager needed game names and save descriptions. `Info.xbx` and `Menu.xbx` disappeared along with the launcher that read them.

But the `Settings` object that parsed those text files was still there. The writable, unsigned `.xbx` extension was still there. The entire modding ecosystem -- config files, title caches, icon databases, skin metadata, Discord presence state -- was built on top of a file format designed for a feature that was cut seven months before the Xbox shipped.

Twenty years of dashboard mods, powered by the ghost of `games.xap`.

## Scripts Microsoft Never Intended: harddrive.xap

`harddrive.xap` is a 1,093-line game launcher and file browser that doesn't exist anywhere in Microsoft's original dashboard. It was built entirely in script by the UIX community.

### The Problem

Microsoft's dashboard had no game launcher. You put a disc in, it played. Game saves lived in the memory manager. There was no "browse your hard drive and pick an XBE to run" feature, because Microsoft never imagined people would have games on the hard drive.

The community needed a launcher. But building one in XAP meant working with only the node types Microsoft provided. There was no `GameBrowser` node, no `FilePicker` node, no `LaunchXBE()` function exposed to scripts. So the community improvised.

### The Hack: Hijacking the Music Player

The music player XIP (`Music_PlayEdit2.xip`) had the visual elements needed for a list-based browser: a scrollable track list with text items, up/down arrows, a selection highlight, and a panel layout. It also had the `MusicCollection` node which could enumerate content from the hard drive.

`harddrive.xap` loads `Music_PlayEdit2.xip` as its archive:

```vrml
DEF theLauncher Level {
    archive "Music_PlayEdit2.xip"
    children [ Inline { url "Music_PlayEdit2/default2.xap" } ]
}
```

It loads the music player's entire visual layout -- the track list panel, the scrolling text items, the selection pod -- and then **repurposes every element**. The track name text nodes (`c.TrackNames.children[i]`) display game titles instead of song names. The selection highlight uses `GameHilite` material instead of the music player's selection style. The `SubMenuHeaderText` shows folder paths instead of album info.

### The Icon Trick

Game icons were the hardest part. Microsoft's dashboard had no general-purpose "load an image from a path" function exposed to scripts. But the `SavedGameGrid` node -- the one Microsoft built for the memory manager -- had `setSelImage()`, which loaded title icons from UDATA on the hard drive. BigJx figured out that you could wire the `SavedGameGrid` as a hidden icon loader:

```javascript
// Set the memory monitor to the hard drive
theMemoryMonitor.curDevUnit = 8;
c.theIconSelector.curDevUnit = theMemoryMonitor.curDevUnit;
c.theIconSelector.isActive = true;
TitleCount = c.theIconSelector.GetTitleCount();

// When highlighting a game, search for its icon by title ID
for (var x = 0; x < TitleCount; x = x + 1) {
    if (c.theIconSelector.GetTitleID(x).toLowerCase() == Icon) {
        c.theIconSelector.curTitle = x;
        c.theIconSelector.setSelImage();  // loads the icon texture
        break;
    }
}
```

The script iterates through every title the `SavedGameGrid` knows about, matches it by title ID, then calls `setSelImage()` to make the node load the icon texture. The icon then appears on a mesh orb in the launcher UI. Microsoft built `setSelImage()` for the memory manager's save browser. The community used it as a general-purpose icon loader.

### The File Browser

The script also implements a full file browser -- something that definitely doesn't exist in Microsoft's XAP node set. It uses a `Folder` object (which Microsoft exposed for the music soundtrack scanner) to enumerate directory contents:

```javascript
var c = new Folder;
c.path = GetDrive(sDrive) + sPath;
// Add folders
for (var i = 0; i < c.subFolders.length(); i = i + 1) {
    CurDirectoryContents[CurDirectoryContents.length] = c.subFolders[i].name;
    sFileType[sFileType.length] = "folder";
}
// Add files
for (var i = 0; i < c.files.length(); i = i + 1) {
    CurFiles[CurFiles.length] = c.files[i].name + "." + c.files[i].type;
}
```

The `Folder` node was meant for finding soundtrack directories. The script uses it to browse the entire hard drive, complete with `..` parent directory navigation, file type detection, and path ellipsis for deep directories.

### Quick Launch Bindings

The script even implements quick-launch keybindings. Hold both triggers and press A, B, X, or Y while highlighting an XBE in the file browser, and it saves the path:

```javascript
if (LeftTrigger & RightTrigger & bInRoot == false) {
    var sPath = arrMenuPaths[nCurLauncherItem0] + "\\" + TempTitleList[nPlayCursor0];
    SetSavedValue("QuickLaunch", "QuickLaunchA", sPath);
    TellUser(Translate("ASSIGNED_QUICKLAUNCH") + "A", "");
}
```

None of this was designed by Microsoft. It's 1,093 lines of script built on top of a music player's visual assets, using a save game browser as an icon loader and a soundtrack folder scanner as a file system browser.

## More Exploits: The Greatest Hits

### The Reboot Button That Isn't

Microsoft's `Recovery` node exists for one purpose: system recovery after a critical failure. The last step of recovery calls `FinishRecovery()`, which reboots the console. The community's take:

```javascript
DEF theReboot Recovery
function Reboot()
// scripted reboot - last stage of a recovery is to reboot
// let's make the box think we just did a recovery and now
// it's time to reboot :)
{
    theReboot.FinishRecovery();
}
```

Create a `Recovery` node. Call `FinishRecovery()`. The Xbox reboots. Microsoft built a disaster recovery system. The community used it as a button.

### config.xap: A Settings Framework in a Language Without Objects

Microsoft's dashboard had settings -- video mode, audio, language, parental controls -- but they were all handled in compiled C++ through the `Config` node. The XAP scripts could call `theConfig.SetVideoMode()` but couldn't add new settings or build new settings UI.

The concept goes back to the tHc era -- early dashboard mods had basic config scripts. UIX Lite cleaned it up and expanded it into `config.xap`, a 1,283-line configuration system that lets users customize everything from main menu layout to Discord presence ports. It implements a data-driven UI framework using parallel arrays as a poor man's struct, because XAP has no objects, no structs, no classes:

```javascript
configList[i] = "Main Orb Style:";        // display label
configValues[i] = "default-MainOrbStyle";  // INI section-key pair
configSelect[i] = "toggleMOS()";           // function to call on A press
```

Three arrays, kept in sync by index. `configList` is the display text, `configValues` encodes the INI file section and key (separated by `-`), and `configSelect` is a function name string that gets called when the user presses A. It's a parallel-array MVC pattern in a scene description language.

The toggle functions cycle through valid values:

```javascript
function toggleMMF() {
    var b = getConfigMenuValue(configValues[LV2Item]);
    if (b == "GoToMemory()") { refreshMenu("GoToMusic()"); }
    else if (b == "GoToMusic()") { refreshMenu("GoToXOnline()"); }
    else if (b == "GoToXOnline()") { refreshMenu("GoToLauncher()"); }
    else { refreshMenu("GoToMemory()"); }
}
```

This configures which *function* each main menu button calls -- the menu buttons are scripted to execute whatever function name is stored in the INI file. You can remap the main menu entirely from the config screen.

The `Settings` object (an INI file reader/writer Microsoft exposed for internal use) does all the persistence:

```javascript
var info = new Settings;
info.file = "Q:\\system\\config.ini";
info.section = "Dashboard Settings";
info.SetValue("Current Skin", SkinMenuList[skinSelect]);
```

It even manages Discord relay port configuration (`togglePorts()` cycles through 1101/1102/1103), quick launch button bindings, which settings menu items are visible, and an "Advanced Mode" toggle that switches the config UI from cycle-through-options to a full keyboard input for typing arbitrary values.

### The Title Cache

Microsoft's dashboard scanned for content at boot and that was it. The community needed a way to cache title lists across reboots -- scanning extended partitions on large hard drives was slow. The solution: abuse the `Settings` INI writer as a database:

```javascript
CacheFile.section = "Cache";
cache = cTitles.join("|");
CacheFile.SetValue(arrMenuNames[x], cache);
```

Title lists are serialized with `|` delimiters, segmented into chunks of 25 (because of string length limits in the INI parser), and stored across numbered keys (`"Games"`, `"Games-1"`, `"Games-2"`). It's a pipe-delimited database built on top of an INI file writer that Microsoft intended for storing simple key-value pairs. But it works, and it's fast.

### The Skin Browser (Theseus Addition)

`skins.xap` is a Theseus-era addition -- the original dashboard had no live skin switching. It loads `Settings_Panel.xip` (Microsoft's settings panel layout) and repurposes it as a skin picker. It uses the `Folder` node (meant for soundtrack directories) to enumerate `Q:\skins\`, displays them in the settings panel's text list, reads skin metadata from unsigned `.xbx` text files, and applies the selection by writing to `config.ini` and calling `theConfig.ApplySkin()` -- a function Theseus added to the `Config` node that didn't exist in Microsoft's original.

### The Main Menu Is Configurable

Microsoft's main menu had four fixed items: Memory, Music, Settings, Xbox Live. UIX Lite's `default.xap` reads button labels and actions from `config.ini`:

```javascript
Button1Text = GetSavedValue("MainMenu", "Button1Text");
Button1Action = GetSavedValue("MainMenu", "Button1Action");
```

Each button can be remapped to Games, Music, Memory, Settings, Xbox Live, Skins, UIX Config, or the Launcher -- all from an INI file. The menu itself is still Microsoft's original 3D panel layout with the same animations and transitions. It just does different things now.

### Extended Partition Detection

The Xbox shipped with 5 partitions. Custom BIOSes added partitions 6-14 for extended storage. The script detects which partitions exist by probing drive letters:

```javascript
var nPartitions = StringToNumber(GetSavedValue("ExtendedPartitions", "Partitions"));
if (nPartitions & 1) RootDirectory[RootDirectory.length] = "F";
if (nPartitions & 2) RootDirectory[RootDirectory.length] = "G";
if (nPartitions & 16) RootDirectory[RootDirectory.length] = "R";
if (nPartitions & 4) RootDirectory[RootDirectory.length] = "H";
if (nPartitions & 8) RootDirectory[RootDirectory.length] = "I";
```

A bitmask stored in an INI file tracks which partitions were found during a scan. This is partition management implemented in a scripting language that was designed for positioning 3D text labels.

### The DVD Dongle Bypass

Microsoft's Xbox DVD playback required a physical IR remote kit -- a USB dongle that plugged into a controller port. Without it, the DVD player refused to work. This wasn't a software limitation. The dongle contained cryptographic keys in its XBE that the DVD playback subsystem validated before allowing MPEG-2 decoding. But the *UI check* was done in XAP script.

Microsoft's `dvd.xap` had this callback:

```javascript
function OnNoDongle()
{
    BlockUser("NoDongle");  // locks the entire DVD UI with an error
}
```

`BlockUser()` put up a full-screen modal message: "You need to connect the DVD Playback Kit receiver to a controller port to watch movies." No dismiss button. You were stuck until you plugged in the dongle or ejected the disc.

The community's fix was one line:

```javascript
function OnNoDongle()
{
    bShowNoDongleMessage = true;  // sets a flag, doesn't block anything
}
```

The UI no longer locks up. The `bRemoteInserted` variable was already initialized to `1` (remote "present") at script startup, so all the remote-check gates throughout the DVD player (`if (bRemoteInserted != 1) return;`) were already passing. The `OnNoDongle` callback was the only thing that could change that state and block the user. By replacing `BlockUser` with a flag, the DVD player just works with a regular controller.

The actual DVD decryption still required proper keys from the XBE side -- this wasn't a complete dongle bypass at the hardware level. But the UI gatekeeping was entirely in script, and script is editable.

### Auto-Launch Toggle

Microsoft's dashboard automatically launched disc content when inserted -- put in a DVD and it started playing, put in a game and it started loading. No option to disable this. The community added a config check:

```javascript
function OnDiscInserted()
{
    var a = GetSavedBooleanValue("default", "AutoLaunchMedia");
    if (a == true) {
        AutoLaunch();
    } else {
        // ask the user first
        var stra;
        if (theDiscDrive.discType == "Audio") {
            stra = Translate("AUTOLAUNCH_AUDIO_ASK");
        }
        // ...show a confirmation dialog
    }
}
```

Another `GetSavedBooleanValue` from the INI config system, wrapped around Microsoft's original auto-launch code. The config.xap settings screen lets the user toggle this on and off.

### Discord Presence via XBE Launch

XAP scripts have no networking API. No sockets, no HTTP, no way to talk to external services. So how does UIX Lite show Discord Rich Presence when you're playing a game?

By launching a separate XBE.

The script writes the game's path to an INI file using the `Settings` object, then calls `launch()` to run `Discord.xbe` -- a separate executable that reads the INI file and sends the presence update over the network. It's inter-process communication via the filesystem, because that's the only tool XAP scripts have:

```javascript
Discord = new Settings;
Discord.file = "C:\\Discord Presence\\shortcut.ini";
Discord.section = "Shortcut";
Discord.SetValue("Path", GetDrivePath(PathArray[2], PathArray[3])
    + Slice.join("\\") + "\\" + launchXbe);
launch("Discord.xbe", "\\Device\\Harddisk0\\Partition2\\Discord Presence");
```

When you return to the dashboard, the initialization code checks if the current Discord state still points to a game. If it does, it overwrites the path with the dashboard's own XBE path and relaunches Discord.xbe to update the presence back to "on the dashboard." The entire presence lifecycle is managed by launching executables and passing data through INI files on disk.

### The Kernel Path Table

Microsoft's retail dashboard mapped three drive letters using NT symbolic links in compiled code. The community needed more -- extended partitions, custom drives, the R: partition. But you can't create NT symbolic links from XAP script. So the script maintains its own parallel mapping of drive letters to raw NT device paths:

```javascript
DriveLocations = new Array("C:\\", "E:\\");
PartitionLocations = new Array(
    "\\Device\\Harddisk0\\Partition2\\",
    "\\Device\\Harddisk0\\Partition1\\"
);

// Extended partitions added dynamically from saved config
if (nPartitions & 1) {
    PartitionLocations[PartitionLocations.length] = "\\Device\\Harddisk0\\Partition6\\";
    DriveLocations[DriveLocations.length] = "F:\\";
}
```

When the script calls `launch()`, it passes the raw NT device path -- `\\Device\\Harddisk0\\Partition6\\Games\\Halo\\default.xbe` -- because that's what the kernel's `XLaunchNewImage` actually needs. The drive letters in the script are just for display. The real work happens in kernel path space, constructed and managed entirely in JavaScript running inside a VRML scene graph.

## Skins

The skin system works by overriding asset paths. When the dashboard loads a texture like `background.xbx`, it checks:

1. `Q:\Skins\{active_skin}\background.xbx` (skin override)
2. `Q:\Xips\default\background.xbx` (default from XIP)

If a skin provides the file, it's used instead of the default. This lets skins change textures, meshes, and even scripts without modifying the base XIP.

The skin format has its roots in the original Xbox modding scene. **tHc Lite** created skins through hex patches and XAP script modifications on the retail XBE. **User.Interface.X** (the original UIX by JbOnE) introduced compiled skins with DCP presets and a material preset system. Theseus reconstructed the MaxMat material engine to be compatible with these existing formats, so the library of community skins works without conversion.
