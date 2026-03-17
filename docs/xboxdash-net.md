# XboxDash.NeT: Two Weeks in September

XboxDash.NeT (also called Xbox Dash Next) sits at a specific point in the timeline: after tHc, tHc Lite, and BSX, but before the Commemorative merge and UIX. It's what happened when JbOnE stopped patching the retail XBE and started building on the source.

tHC Final was the last purely hex-patched release. Then source access happened, and the tHc XIPs got adapted to run on a custom-compiled engine with real C++ objects backing the script functions. Xbox Dash Next is the compiled build of that work -- the bridge between tHc and what would become UIX.

## Reading the Timestamps

The source archive tells the story through file dates. Every file from the Xbox source leak is stamped `2002-02-01 19:50`. JbOnE's modifications are dated August 26 through September 12, 2004. Two weeks.

**Untouched from the leak:**
- The script VM (`Compiler.cpp`, `Lexer.cpp`, `Parser.cpp`) -- already worked fine
- Visual effects (`Background.cpp`, `DeltaField.cpp`, `StarField.cpp`) -- no changes needed
- Game copy system (`CopyDest.cpp`, `CopyGames.cpp`) -- kept as-is

**JbOnE's changes (50+ files in two weeks):**
- `config.cpp` -- configuration system
- `MaxMat.cpp` (66KB) -- material and skin system
- `Joystick.cpp` -- input
- `Harddrive.cpp` -- new HardDrive node
- `TitleMenu.cpp` -- new TitleMenu node
- `Xbox.cpp` -- drive mappings
- `main.cpp`, `xapp.h` -- initialization
- `dvd2.cpp` (94KB) -- DVD player
- `Runner.cpp`, `Node.cpp` -- VM extensions
- `SavedGameGrid.cpp` -- memory management
- `xbe.cpp` -- XBE reader

The `lib/` folder has Xbox debug SDK libraries -- he was compiling with full debug symbols.

## Three Drives Became Thirteen

The biggest change was in `Xbox.cpp`. Microsoft mapped three drive letters:

```cpp
// Microsoft: 3 drives
IoCreateSymbolicLink("\\??\\C:", "\\Device\\Harddisk0\\partition1");
IoCreateSymbolicLink("\\??\\Y:", "\\Device\\Harddisk0\\partition2");
IoCreateSymbolicLink("\\??\\T:", "...\\TDATA\\fffe0000");
```

JbOnE mapped thirteen:

```cpp
// JbOnE: 13 drives
C: -> partition2         // system
D: -> CdRom0             // DVD drive
E: -> partition1         // user data
F: -> partition6          // extended
G: -> partition7          // extended
X: -> partition5          // cache
Y: -> partition4          // cache
Z: -> partition3          // cache
T: -> TDATA              // title data
U: -> UDATA              // user save data
MUSIC: -> TDATA/fffe0000  // dashboard audio temp
```

He also had to delete the kernel's existing D: mapping before remapping it -- one line of NT kernel API:

```cpp
OBJECT_STRING ddd = CONSTANT_OBJECT_STRING("\\??\\d:");
IoDeleteSymbolicLink(&ddd);
```

The `MUSIC:` drive was his own addition -- a dedicated mount point for the dashboard's temporary audio files, keeping them separate from the general TDATA directory.

## The Header Changes

`xapp.h` shows the surgical additions:

```cpp
//jbone
HRESULT CXApp::InitAudio();

TCHAR* m_sFontDir;
TCHAR* m_sXipDir;
TCHAR* m_sSkinDir;

bool f_exists;
bool g_exists;
```

Configurable paths for fonts, XIPs, and skins -- no longer hardcoded to Microsoft's retail locations. Extended partition flags for F: and G: drives. And the `MAX_BLOCKS_TO_SHOW` bumped from 50,000 to 500,000,000 -- because modded consoles had hard drives that Microsoft never planned for.

The `//jbone` comment is the only signature in the source.

## The Compiled Build

Xbox Dash Next is what this source produced. The build shipped with:

- **7 color presets**: MS Dash - Blue, B&W, Green, Orange, Purple, Red, Yellow
- **Adapted tHc XIPs** running on the new engine
- **Network config**: Static IP, FTP server with login/password
- **Background music on boot**
- **Configurable main menu** with password-protected buttons

The `default.xbx` config file shows the architecture:

```ini
[Main Menu Orb]
OrbType=tHc

[Main Menu Button 2 Settings]
Launch=GoToMusicPlayWithSubs()
MenuType=Xbox Hard Drive

[Games Menu]
Name=Games
Path=games
```

`OrbType=tHc` -- this is a tHc derivative running on a custom engine. The hard drive menu still uses `GoToMusicPlayWithSubs()` -- the music player hijack adapted from tHc's scripts. Some things carry forward even when you rebuild from source.

## JbOnE's Signature

Line 2062 of `default.xap`:

```javascript
////// All functions I added are below here :)   - JbOnE
```

Below that line: test code for the new engine APIs he'd just built:

```javascript
// Testing HardDrive file operations
var test = theHardDrive.CopyDirectory("F:\\storage", "E:\\storage");

// Network info display
TellUser("Xbox IP: " + theXboxNetwork.GetXboxIP() + "\n" +
         "Subnet Mask: " + theXboxNetwork.GetXboxSubnet() + "\n" +
         "Gateway: " + theXboxNetwork.GetXboxGateway() + "\n" +
         "Link Status: " + theXboxNetwork.GetLinkStatus() + "\n" +
         "Speed: " + theXboxNetwork.GetLinkType(), "");

// Drive space
var a = theHardDrive.GetFreeSpace("E:\\");
var b = theHardDrive.GetTotalSpace("E:\\");
```

You can see him testing each new API as he built it. `CopyDirectory` to verify file operations work. Network info to check the TCP/IP stack. Drive space queries. This is a developer's working build with test code still in it.

## The Skin System (Partial)

`MaxMat.cpp` reads 149 material colors from a skin `.xbx` file at boot:

```cpp
SkinXBX.Open("E:\\UDATA\\XboxDash\\Xbox.xbx");
SkinXBX.GetValue("Dashboard Settings", "Current Skin", CurrentSkinFile, MAX_PATH);
SkinXBX.Open("A:\\Skins\\" + CurrentSkinFile + "\\" + CurrentSkinFile + ".xbx");

SkinXBX.GetValue("InnerWall_01", "ColorA", ColorA, MAX_PATH);
SkinXBX.GetValue("InnerWall_01", "ColorB", ColorB, MAX_PATH);
// ... 147 more material reads
SkinXBX.GetValue("Typesdsafsda", "Color", ColorA, MAX_PATH);
```

Boot only. No runtime switching. The 7 skin presets work because the colors are read once at startup. Want a different skin? Change the config and reboot. The live-switching came later in UIX proper.

And yes, `"Typesdsafsda"` is in there on line 1077. Someone mashed their keyboard for a material name and it's been in every version of this codebase for 20 years.

## JbOnE's Toys

Alongside the source, a separate archive (`jbonestoys.rar`) contained the tools he used:

- **Team Assembly XKUtils** (GPL): XKEEPROM, XKCRC, XKSHA1, XKRC4, XKHDD, XKUtils -- the Xbox crypto and EEPROM library
- **AV encoder registers**: `av_register_final.h` and `modeset.c` for video output mode switching

The XKUtils code traveled from Team Assembly through JbOnE's toolkit into XboxDash.NeT, then into UIX, and eventually into Theseus. GPL open source that's been in continuous use across the Xbox scene for over 20 years.

## Where This Fits

```
tHc -> tHc Lite -> BSX (fork)
                      \
tHC Final -----------> XboxDash.NeT / Xbox Dash Next (source-based)
                                          |
                            Commemorative merge (tHc + BSX scripts on this engine)
                                          |
                                         UIX
```

Xbox Dash Next is the moment the dashboard stopped being a patched retail binary and became its own thing. The tHc XIP heritage carried forward in the scripts. The engine underneath was new. Everything that came after -- the Commemorative merge, UIX, UIX2, and eventually Theseus -- built on this foundation.
