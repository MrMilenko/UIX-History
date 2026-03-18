# The Skin System: How It Works

The technical side of how dashboard skins work -- the invisible mesh preloader trick, the texture lookup pipeline, and how colors and textures get swapped. For the full historical progression from tHc through Theseus, see **[The Lineage](lineage.md#skinning-from-hex-editing-to-live-switching)**.

## The Early Days: tHc and BSX (2002-2004)

The first "skins" were texture replacements packed into modified XIPs. Replace the textures in a XIP, resign it (or patch out the signature check), and the dashboard looks different. tHc Lite built tools that patched and resigned the dashboard XBE, handling both color overrides and bogus XIP signatures. Later "no-resign" patches removed the XIP signature check entirely, which made XIP modifications much easier to distribute.

The key file was `skin.xip` -- a XIP containing replacement textures and a `skin.xap` script that told the engine what to load. Inside, a small mesh (`skin.xm` -- 28 vertices, barely a rectangle) served as a dummy geometry to attach textures to.

BSX and tHc both used this approach. The textures covered every visible surface: cellwall, menu highlights, DVD buttons, background panels, Xbox logos, music player controls. Swap the textures in the XIP, change the look.

## The Trick: Invisible Preloading

The `skin.xap` script reveals how the preloader works:

```vrml
DEF xbox4 Transform
{
    children [
        Shape {
            appearance Appearance {
                texture ImageTexture { url "xbox4.tga" }
            }
            geometry Mesh { url "skin.xm" }
        }
    ]
    translation 0 0 300
}
```

Every texture gets mapped onto `skin.xm` and positioned at `translation 0 0 300` -- 300 units behind the camera, completely invisible. The engine loads these as real scene objects with real textures bound to real geometry. The textures get cached in the engine's texture system.

When the dashboard later tries to load `xbox4.xbx` for the main menu, `FindObjectInXIP()` checks the skin directory first. The preloaded texture from skin.xip is already in memory. The skin texture wins.

It's a texture preloader disguised as an invisible 3D scene. The `skin.xm` mesh exists only to give the textures something to attach to -- a 28-vertex dummy that makes the XAP parser happy.

## UIX's Skin System (2004-2005)

JbOnE's User.Interface.X took the concept further with a proper skin picker UI. UIX's skin system had around 149 material name mappings and a menu (`skins.xap`) for browsing and selecting installed skins. Select a skin, the dashboard restarts with the new textures loaded.

The skin picker read skin metadata from `.xbx` text files -- author name, version, description -- using the same `Settings` object exploit that powered everything else in the modding scene.

## UIX Lite: The Patched 5960 (2020-Present)

UIX Lite is a patched retail 5960 XBE -- same approach as tHc, but for the final dashboard version. It's what people actually run on their Xboxes.

The skin system on UIX Lite is closer to the original tHc approach: texture replacements via `skin.xip`, with the same invisible mesh preloader trick. But the original tHc/UIX skin systems also patched color values at specific hex offsets in the XBE -- menu highlight tint, glow intensity, text colors. On 5960, those offsets are different and the approach doesn't carry over cleanly.

Rocky5 solved it with the [Xboxdash 5960 Colourizer](https://github.com/OfficialTeamUIX/UIX-Lite/tree/main/Tools/Colour%20Patcher) -- a Python tool that patches 80+ color values directly in the 5960 XBE via hex offsets. Skin authors define color schemes in `.ini` files (RGB or ARGB hex codes), and the tool writes them into the binary. Six presets ship with the tool, including one that matches the original dashboard colors. It also handles extended partition patches and DVD region compatibility in the same pass.

For textures, the [Xip Skin maker](https://github.com/OfficialTeamUIX/UIX-Lite/tree/main/Tools/Xip%20Skin%20maker) creates `skin.xip` files from folders of replacement images -- drag a skin folder onto a batch file, get a skin.xip. PSD templates included. Textures get resized automatically. FTP support for pushing directly to an Xbox. Special thanks to Voltaic for pixit, which the tool builds on.

The full workflow: Colourizer patches colors in the XBE, Skin maker packs textures into skin.xip, and both get deployed to the console. Colors and textures, two separate tools, one skinned dashboard.

## Theseus: The Hybrid (2020-Present)

Theseus is the rebuilt engine -- the decompilation that became its own thing. (It was called "UIX Lite" internally for a while, which caused confusion. Two projects, same name. That's why it got renamed.)

Theseus's skin system is a Frankenstein of three projects:

- **tHc Lite's `skin.xip`** -- the preload mechanism (invisible mesh + texture cache)
- **XboxDash.NeT's material definitions** -- the material name mappings that connect skin textures to dashboard surfaces
- **UIX's skin picker** -- the selection UI (adapted into `skins.xap`)

Two XIPs handle different jobs:
- **`skin.xip`** = the preloader. Loads skin textures at boot from the current skin folder (e.g., `Q:\Skins\Stock\`). Runs before the main menu renders.
- **`skins.xip`** = the skin selection menu. Browse installed skins, pick one, restart.

The skin preloader works on Theseus because the C++ `FindObjectInXIP()` function checks the skin directory before looking in XIPs -- the same lookup order that made the original tHc trick work. `MaxMaterial` nodes in the scene graph resolve texture names through this pipeline, so any texture in the skin folder automatically overrides the matching XIP texture.

### How Material Lookup Works (from the binary)

When a `MaxMaterial` node's `name` property is set in a XAP script, the engine calls `LookupMatInfo()` (at `0x0005A130` in the 5960 binary). This function resolves a material name string to its rendering properties using a binary search over the sorted `g_rgMatInfo` array:

```asm
; LookupMatInfo -- called when a MaxMaterial "name" is set
0005A133: MOV EAX, [ESI + 0xA0]            ; cached matinfo pointer
0005A139: TEST EAX, EAX                     ; already looked up?
0005A13B: JNZ  0x0005A17D                   ; yes, skip to setup

; First lookup: binary search the global material table
0005A13D: MOV EAX, [ESI + 0x90]            ; material name string
0005A147: MOV ECX, [0x0017E754]            ; g_nMatInfoCount
0005A14D: PUSH 0x57C90                      ; compare function
0005A152: PUSH 0x4                          ; element size (4-byte pointer)
0005A154: PUSH ECX                          ; count
0005A155: PUSH 0x17C6B8                     ; g_rgMatInfo array base
0005A15A: PUSH EAX                          ; search key (material name)
0005A15B: CALL 0x00074040                   ; bsearch()
```

After the match, the function checks if the material is a tube variant -- tubes use a different falloff shader mode:

```asm
; Check tube materials for shader variant selection
0005A1B2: PUSH 0x2917C                      ; "Tubes"
0005A1B7: PUSH EDX                          ; material name
0005A1B8: CALL 0x00073412                   ; wcscmp
0005A1C0: TEST EAX, EAX
0005A1C2: JZ   0x0005A216                   ; match -> set tube shader flag
0005A1CA: PUSH 0x29138                      ; "TubesFade"
        ...
0005A1E2: PUSH 0x29128                      ; "TubesQ"
        ...
0005A1FA: PUSH 0x2910C                      ; "Tube"
```

This is the pipeline that makes skins work: material names are resolved by binary search against a sorted global table (sorted by `qsort` during `Material_Init`), and the resolved material info controls the rendering properties. A skin overrides the texture bound to a material name, but the material TYPE (solid, falloff, aniso) comes from this compiled lookup table -- which is why getting the material types right matters.

## On Desktop

UIX Desktop's skin system is the same pipeline -- `FindObjectInXIP()` checks `Q:\Skins\Stock\` first, then falls back to the XIP contents. The Stock skin folder contains `.xbx` textures that override the XIP defaults. This is how alpha dashboard builds from 2001 can render with modern skin textures: the alpha XIPs provide the geometry and base materials, the Stock skin folder overrides the textures where material names match.

The `skin.xip` preloader still works on desktop -- the invisible `skin.xm` mesh trick runs the same way it did on Xbox hardware. Twenty years of compatibility for an invisible rectangle.
