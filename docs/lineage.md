# The Lineage: From Microsoft to UIX Lite

How the Xbox dashboard modding scene evolved through rivalry, collaboration, and a lot of copy-pasting between groups that didn't always get along.

## The Chain

```
Microsoft retail XAPs (2001-2005)
    |
    v
tHc / tHc Lite (Gcue, others)
    |-- Patched retail XBE + custom XAP scripts on top
    |-- Added: submenu system, background music, setup scripts
    |-- Base: dashboard version 4920
    |
    +---> BSX / BlackStormX (BobMcGee, acidbath, ZogoChieftan)
    |     |-- Forked from tHc after a split
    |     |-- Rebuilt visuals using REZN8/Seton Kim design concepts
    |     |-- Added: config panel, skins, LED control, in-dash hex editing
    |     |-- "circumstances forced us to form our own group"
    |
    v
tHC Final (JbOnE)
    |-- Last hex-patched retail XBE release
    |
    v
XboxDash.NeT / Xbox Dash Next (JbOnE, source-based)
    |-- Got source access, adapted tHc XIPs to run on custom-compiled engine
    |-- Two-week sprint (Sept 2004): 50+ files modified, 13 drive mappings
    |-- Added engine objects: TitleMenu, HardDrive, XboxNetwork
    |-- Partial skin system (149 materials, boot-only)
    |-- See [XboxDash.NeT: Two Weeks in September](xboxdash-net.md)
    |
    v
Commemorative Build ("The Merge")
    |-- tHc + BSX buried the hatchet
    |-- Merged both scripts onto JbOnE's source-based engine
    |-- Added: DNA backgrounds, 874-line file manager, favorites, config system
    |
    v
UIX / User.Interface.X (JbOnE)
    |-- Full evolution of the XboxDash.NeT engine
    |-- Custom XBE with expanded API: TitleMenu, HardDrive, XboxNetwork, Settings
    |-- 1,820-line hand-written API documentation
    |-- UIX2 planned but shelved after internal leak
    |
    v
UIX Lite (TeamUIX, 2020-present)
    |-- Named as a throwback to "tHc Lite" -- not UIX proper, but a lighter take
    |-- Ported commemorative/UIX scripts to retail 5960 dashboard
    |-- Adapted commemorative/UIX scripts, rebuilt what was possible as XAP hacks
    |-- Some features couldn't be replicated due to retail XBE limitations
    |-- Key workarounds:
    |     - TitleMenu.LaunchTitle() --> music player hijack + Folder nodes
    |     - HardDrive.ExecuteFile() --> launch() with kernel device paths
    |     - TitleMenu.GetTitleID() --> SavedGameGrid icon loading trick
    |     - XboxNetwork --> scripted via Settings INI
    |     - Config panel --> parallel array MVC framework
    |-- Compatible with Insignia (Xbox Live revival)
    |
    v
Theseus (Milenko)
    |-- Reverse-engineered XBE replacement
    |-- Re-added engine objects (ApplySkin, etc.) that JbOnE's XBE had
    |-- XIPs run on both Theseus and retail 5960
    |
    v
UIX Desktop
    |-- Theseus engine on macOS/Linux/Windows via SDL2+OpenGL
    |-- Same XIPs, same scripts, different platform
```

## The tHc / BSX Split

tHc's lineage started with the True Blue and XboxDash-Hacked patchers (pre-patched retail XBEs with color changes and basic mods), then evolved into tHc proper, and then tHc Lite -- each iteration adding more XAP script functionality. Gcue wrote the submenu system that became the template for every community dashboard menu.

BSX (BlackStormX) forked from tHc after internal disagreements. From BSX's readme:

> *Initially, this project was meant to be an extension of thc's groundbreaking work, however, circumstances forced us to form our own group. While almost all thc code has been removed from this dash, we would like to give credit to thc for the following functions: AutoAdd, Skins archieving from skin.xip, Hard Drive Space (hack by fuck_db and implementation initially by thc)*

BSX rebuilt the visual identity using concepts from Seton Kim's REZN8 design work -- the DNA helix backgrounds, the different orb styling. They added an in-dash config panel, skin switching, LED control, and an XBE hex patcher. ZogoChieftan's color patcher let you change the dashboard's color scheme without modifying XIPs.

The BSX readme ended with: *"If, however, you don't like our dash please play hide-and-go-fuck-yourself."*

## The Merge

The commemorative XIP set was built as a peace offering. The header of `default.xap` says:

> *This xip set is not an official uix release. It is strictly intended as a gift to our supporters over the past few years and as a celebration of the end of bickering between THC and BSX. "The Merge" Commemorative Xip Set v1.0b*

It combined tHc's scripting with BSX's visual assets -- including three selectable background meshes (`DNA.xap`, `CHAINS.xap`, `SPIRALS.xap`), a dual-pane file manager, a favorites system, and the first version of the config.xap settings framework.

The code shows the merge in practice. From `favs.xap`:

```javascript
// copied from the bsx ms style hd menus... most code is not needed for skins
// copied from skins, now being used for favs also :/
```

Scripts borrowed from BSX, adapted from the skin system, repurposed for favorites. This is how community codebases evolve -- not through clean architecture, but through copy-paste, adaptation, and comments apologizing for the mess.

## The parseTrans Hack

One detail worth preserving: the commemorative config.xap contains a function for parsing 3D translation vectors that's still in UIX Lite today. XAP had no way to convert a Transform's translation to individual X/Y/Z values, so they wrote this:

```javascript
function parseTrans(a, b)
{
    var inTran = returnString(a);
    // parse "X Y Z" string into components
    var parsy1 = inTran.substr(0, inTran.indexOf(" "));
    // ...
}

function returnString(x)
{
    theIntToString.children[0].geometry.text = x;
    var b = theIntToString.children[0].geometry.text;
    return b;
}
```

The `returnString` function writes a value to a hidden Text node's `text` property, then reads it back as a string. XAP had no `toString()` for vectors or numbers, so they used the text rendering system as a type converter. A hidden, invisible Text node serving as a cast operator. This hack has survived 20 years of dashboard development because nobody found a better way to do it.

## JbOnE's API

When JbOnE built the UIX XBE, he added engine objects that the commemorative scripts could call directly:

```javascript
theHardDrive.ExecuteFile("e:\\somepath\\somefile.xbe");
theGamesMenu.LaunchTitle(5);
theXboxNetwork.StartFTPServices();
theConfig.ChangeSkin("blue");
```

These were compiled C++ functions exposed to the script VM. When UIX Lite moved to the retail 5960 dashboard, none of these existed. Every one had to be reimplemented using the scripting hacks documented in the [XAP exploits](xap-scripting.md):

| JbOnE's XBE had | UIX Lite replaced it with |
|------------------|--------------------------|
| `TitleMenu.LaunchTitle()` | Music player hijack + `launch()` with kernel paths |
| `TitleMenu.GetTitleName()` | Title cache in pipe-delimited INI files |
| `HardDrive.ExecuteFile()` | `launch()` with constructed `\\Device\\` paths |
| `HardDrive` file operations | `Folder` nodes meant for soundtrack browsing |
| `XboxNetwork.*` | Settings INI for config, separate XBE launch for Discord |
| `Config.ChangeSkin()` | `theConfig.ApplySkin()` (added back in Theseus) |

## Skinning: From Hex Editing to Live Switching

The skin system's evolution mirrors the entire project's arc -- from destructive binary patching to data-driven runtime switching.

**tHc era**: Colors were hex-edited directly into the XBE binary. Pick a color, find the bytes, patch them. Permanent, destructive, one skin at a time. If you wanted to switch back, you needed a backup of the original XBE.

**BSX**: ZogoChieftan built an in-dash color patcher -- hex editing from inside the dashboard itself. Skin files lived in `X:\Skins\` and were selected through the config panel. Still XBE-level patching, but at least you didn't need a hex editor.

**XboxDash.NeT (JbOnE's intermediate)**: A partial skin system. `Material_Init()` read skin colors from an `.xbx` file at boot -- 149 material color reads from `A:\Skins\{name}\{name}.xbx`, with the current skin name stored in `E:\UDATA\XboxDash\Xbox.xbx`. But no runtime switching. Load at boot, done. Want a different skin? Edit the file and reboot. The code shows the foundation being built -- all the material names are there (`"Typesdsafsda"` included), but the live-switching plumbing wasn't wired up yet.

**UIX (JbOnE)**: The XBE got a `ChangeSkin()` engine function. Skins became `.xbx` text files (unsigned INI format) with per-material color definitions:

```ini
#   Xbox Dash Skin Material Data
#   Skin Name - MS Dash - Green

[InnerWall_01]
ColorA=243, 255, 107, 192
ColorB=20, 192, 0, 0

[NavType]
ColorA=125, 198, 34, 100
```

Each section name maps to a MaxMaterial in the XAP scene graph. `ColorA` and `ColorB` define the two-tone gradient that the falloff shader applies. A skin folder also contains replacement textures (`.xbx` binaries) and meshes (`.xm` files) that override the defaults. Live switching -- change skins without rebooting.

**UIX Lite (retail 5960)**: The retail XBE has no `ChangeSkin()` function. Skin presets still work for textures and meshes (the file override system checks `Q:\Skins\{name}\` before loading from the XIP), but the material colors are baked into the XBE. Rocky5's Colourizer tool patches the color values in the XBE binary one time. To change color schemes, you re-patch the XBE.

**Theseus**: `ApplySkin()` re-added as an engine function. Live skin switching restored. The MaxMat system reads the skin `.xbx` file and updates every material's colors at runtime -- `ReloadSolidMat`, `ReloadFalloffMat`, `ReloadModulateMat` iterate through the material table and apply the new values.

The material names tell their own story. `"Typesdsafsda"` appears in BSX's code from 2004 as a material name for deselected menu items. It's still in UIX Lite's `harddrive.xap` today. Someone mashed their keyboard for a placeholder name 20 years ago and nobody ever changed it.

The commemorative scripts were the Rosetta Stone. They showed what the features were supposed to do. The retail 5960 dashboard showed what tools were available. UIX Lite bridged the gap.
