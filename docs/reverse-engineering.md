# Reverse Engineering the Xbox Dashboard

## The Dashboard

The original Xbox ran a stripped-down Windows 2000 kernel on standard x86 hardware: a Pentium III, an NVIDIA NV2A GPU (basically a GeForce 3), a standard IDE hard drive. The OS booted from flash into a dashboard application (`xboxdash.xbe`) that handled everything the user saw: game launching, save management, audio CD ripping, DVD playback, system settings, and eventually Xbox Live.

Microsoft built it as a full 3D application on Direct3D 8, but with a twist: almost the entire UI was defined in script files, not compiled C++. The scripts used a custom language based on VRML97 (a 3D scene description format from the late '90s) extended with JavaScript-like logic. Those scripts were packed into archive files called XIPs, and the dashboard loaded them at runtime. Change a script, change the UI. No recompilation needed.

Microsoft never documented the script format, the archive structure, or the binary texture format. The XDK told you how to write games, not how to modify the dashboard. But they made some choices that left the door wide open.

## The Door

### The .xbx Extension

The `.xbx` format was Microsoft's binary texture container -- DXT-compressed pixel data with an XPR0 header for the GPU to consume directly. But somebody reused the `.xbx` extension for plain text metadata files too. `TitleMeta.xbx` and `SaveMeta.xbx` were UTF-16LE text sitting right next to actual textures in the UDATA directory. And crucially: **none of these files were signed.** You could write whatever you wanted to UDATA and TDATA, and the dashboard would read it.

This extension originally existed for the [game launcher that was cut before retail](alpha-dashboard.md) -- `Info.xbx` for news feeds, `Menu.xbx` for custom menus. The launcher died, but the writable unsigned text files lived on. The entire modding ecosystem was built on the ghost of a feature Microsoft killed seven months before the console shipped.

### XIP Signatures

XIP archives were signed. The dashboard verified HMAC signatures over 64KB chunks of each archive before loading scripts. But the check was a single branch instruction in the XBE, and once modchips and softmods let you run unsigned code, patching it out was trivial.

Microsoft caught on -- with each dashboard update, they moved the signature check to a different memory address. The community would patch it out, Microsoft would move it, the community would find it again. Cat and mouse over a single branch instruction.

### XAP Scripts

The XAP scripting format turned out to be readable VRML97 extended with JavaScript-like logic. The syntax was close enough to VRML that anyone who'd seen a `.wrl` file could read an XAP. The scripts defined everything: 3D transforms, materials, textures, animations, input handlers, state machines. And since XAP files were compiled at load time from plain text, you could edit them with a text editor and see changes immediately.

This is what made the modding scene possible. You didn't need to reverse-engineer the C++ engine to build custom dashboards -- you just needed to understand the script system well enough to write your own scenes.

The Xbox had replacement dashboards -- EvolutionX, UnleashX, XBMC -- that worked fine as launchers. But a lot of people gravitated toward dash mods instead. The reasoning was simple: Microsoft had a billion-dollar company behind that dashboard. Seton Kim's REZN8 designs, professional 3D artists modeling every pod and mechanical arm, audio engineers processing Apollo mission recordings into ambient soundscapes, custom vertex shaders for the NV2A. Why throw all that away for a 2D list menu when you could hack the real thing?

## The Community (2001-2005)

The Xbox modding scene was building custom dashboards almost immediately. **tHc** and **tHc Lite** hex-patched the retail XBE directly: changing color codes, file paths, DVD region signatures. More advanced work involved binary expansion and ASM injection. Every patch taught the community something about the dashboard's internals.

**User.Interface.X** was JbOnE's project -- a dashboard modification with custom XIP/XAP scripts and a skin system. When BlackStormX and tHc merged, they formed **TeamUIX**. The community built dozens of skins, pushed XAP scripting to its limits, and created tools for editing and repacking XIPs.

The community settled on dashboard version **4920** as the standard base. It was stable, well-understood, and had Xbox Live support -- but the community never used Live, so the custom XIPs and scripts were built without it in mind. UIX replaced stock scripts with community versions that didn't touch the Live integration. Later builds (5659, 5960) added improved Live features, DLC management, and a modern memory UI. By the mid-2000s, community dashboards looked great but couldn't do everything a stock 5960 dashboard could.

The scene accepted this tradeoff. Xbox-Scene had an unwritten rule: **stay off Xbox Live.** Modding was for homebrew, backups, and customization.

Then the scene went quiet.

## The Revival (2020)

Most people forgot what TeamUIX did. The projects were from 2003. The people had moved on. The releases floating around were broken builds nobody maintained.

But some of us remembered. During COVID -- stuck at home, going through a divorce, looking for something to hold onto -- the Xbox modding scene became that thing. It had been a safe space as a kid, sitting in IRC chatrooms watching people build things. Xbox-Scene relaunched as a Discord server and people who'd been around in 2003 started finding each other again.

With all that free time and all that nostalgia, a question finally had room to breathe: *What can we make work on the final 5960 dashboard, just using XIP/XAP modifications?*

headph0ne handed over a massive archive -- UIX builds, BSX, tHc, Commemorative XIPs, skins, development files he'd been sitting on for almost two decades. The commemorative XIP set turned out to be useful beyond the scripts themselves. It showed how the UIX team had solved problems in XAP that Microsoft solved in C++. Comparing the scripted approach against the compiled 5960 approach helped with decompilation -- you could see what a feature was *supposed* to do from the script side, then find the corresponding code in the binary.

The goal was to get community XIPs working on the **final retail 5960 dashboard** -- the version with full Xbox Live support, DLC management, modern memory UI, everything. This meant understanding every XBE patch the community had developed for 4920, figuring out what each one did, and applying the equivalent to 5960. Scripts that assumed 4920's simpler node types needed to handle 5960's newer features. What couldn't be patched in the XBE got replicated in XAP script.

At the same time, **Project Insignia** was bringing Xbox Live back. The old "stay off Live" rule didn't apply anymore -- the service itself was being resurrected. And now there was a dashboard mod that actually ran on 5960, the version Insignia was targeting.

That work became **UIX Lite** -- a patched 5960 XBE with community XIPs, Xbox Live compatible, running on the final retail dashboard.

## The RE Work

### Phase 1: Scripts and XIPs

The first focus was understanding the XAP/XIP system from the outside in. What did the scripts actually do? How did the archive format work? What node types existed and what properties did they expose?

The XIP format was approachable -- `XIP0` magic number, a directory of file entries, and packed data blobs. The community's existing tools (Voltaic's pixit, MaTiAz5's UnXiP, and others) had already documented much of this. Extracting scripts from XIPs and reading them was straightforward.

A massive head start came from JbOnE's [dash syntax reference](jbone_dash_syntax_2005.txt) -- 1,820 lines of hand-written API documentation covering node types, properties, functions, and scripting patterns. It wasn't complete, and some of it had been modified for UIX's purposes rather than documenting Microsoft's original behavior, but it was the closest thing to documentation the script system ever had. Between the extracted XAP files, JbOnE's reference, and the community's accumulated knowledge from years of XIP modding, a working picture of the script system came together.

The harder part was understanding the relationship between the scripts and the engine. XAP scripts referenced node types (`Joystick`, `Level`, `Inline`, `SavedGameGrid`, `TitleCollection`) that were implemented in C++. The scripts called properties and functions on those nodes by name. To modify the dashboard confidently, you needed to know what the engine did when a script set `theSavedGameGrid.curDevUnit = 8` or called `theConfig.SetLanguage(3)`.

### Phase 2: Textures

Once the scripts were understood, the next question was how assets loaded. Textures came first because they were the most visible -- change a texture and you can see it immediately.

XBX textures used standard XPR0 headers followed by DXT-compressed data. The compression was standard DXT that any PC GPU could handle, but the pixel layout was swizzled -- Morton-coded Z-order curve, the same tiling pattern the NV2A GPU used for memory access optimization. Figuring out the swizzle pattern was the main hurdle. Once unswizzled, the data was standard DXT1/DXT3/DXT5.

The texture pipeline went: XBX file on disk -> XPR0 header parsed -> pixel data unswizzled -> D3D texture created -> bound to materials in the scene graph. Understanding this pipeline meant understanding how `FindObjectInXIP()` resolved texture references, how the skin system overrode XIP textures with loose files, and how `MaxMaterial` nodes connected script-side material names to engine-side texture objects.

### Phase 3: Meshes

Meshes were next. XM files turned out to be straightforward vertex/index buffer dumps -- a header with vertex count, index count, FVF (Flexible Vertex Format) flags, and stride, followed by raw vertex data and raw index data. The FVF told you what each vertex contained: position, normal, texture coordinates, diffuse color, in what combination.

Inside XIPs, meshes were stored differently. Individual XM files got split into shared index buffers and vertex buffers, with mesh reference entries pointing into those buffers by offset. A single XIP could pack dozens of meshes into one IB/VB pair, with each named mesh reference selecting a range of indices. This was Microsoft's optimization for reducing draw call overhead -- fewer buffer switches.

The mesh rendering pipeline connected to Direct3D 8's fixed-function pipeline: `SetStreamSource()` for vertex buffers, `SetIndices()` for index buffers, `DrawIndexedPrimitive()` for the actual draw. The FVF flags determined which vertex shader to use. The `falloff` property on certain meshes triggered a custom vertex shader (`effect3.vsh` -- found on a recovery disc from build 3729) that computed viewing-angle transparency for the X-ray glass effect.

### Phase 4: The Engine

The XBE did the heavy lifting -- D3D8 rendering, kernel calls, input handling -- but the meat of the dashboard was in the XIPs. The scripts and assets defined what you saw and how you interacted with it. The XBE was just the machine that ran them.

Most of the RE time was spent in the XIPs, reading scripts, tracing how node properties mapped to visual changes, figuring out the asset loading pipeline. The XBE work came in when you hit a wall: a script sets `theSavedGameGrid.curDevUnit = 8` -- what does the C++ actually do with that? A texture URL references `"cellwall.xbx"` -- how does the engine find it, unswizzle it, and hand it to the GPU?

The 5960 XBE went into both IDA Pro and Ghidra. XboxDev's [ghidra-xbe](https://github.com/XboxDev/ghidra-xbe) extension was essential -- it handles XBE loading and integrates symbol recovery from XbSymbolDatabase, which meant known Xbox SDK functions got identified automatically. Between IDA's decompiler and Ghidra's free-form analysis, different subsystems got tackled in whichever tool handled them better.

The approach was practical: inject code blocks into the binary to test changes. Hard drive path modifications. Color code overrides. XIP signature bypass. Each change validated understanding of a specific subsystem, and the working patches became the foundation for reconstructed C++ code.

The node reflection system was the key that made the XBE tractable. Every node class registered itself with the script VM through macro tables:

```cpp
IMPLEMENT_NODE("Joystick", CJoystick, CNode)

START_NODE_PROPS(CJoystick, CNode)
    NODE_PROP(pt_boolean, CJoystick, isBound)
    NODE_PROP(pt_number,  CJoystick, xaxis)
    NODE_PROP(pt_number,  CJoystick, yaxis)
END_NODE_PROPS()

START_NODE_FUN(CJoystick, CNode)
    NODE_FUN_VV(OnMoveUp)
    NODE_FUN_VV(OnADown)
END_NODE_FUN()
```

These macros embedded property names, type information, and member offsets as string literals in the binary. Once you found the pattern for one node class, you could find them all. The macro tables acted like a manifest: every string the script VM could resolve, every C++ member it mapped to.

Class names came from error messages and debug strings baked into the XBE (`"Error in SavedGameGrid::selectUp"`, RTTI structures, vtable layouts). The `IMPLEMENT_NODE` macros embedded type names as literals. Between the node tables, error strings, and RTTI, enough of the class hierarchy was recoverable to reconstruct compilable C++.

Modern RE tooling made an enormous difference. What would have taken months with a hex editor in 2003 could be done in days with IDA's decompiler and Ghidra's analysis in 2020. The ghidra-xbe extension handled the Xbox-specific parts -- XBE headers, kernel imports, SDK symbol matching -- so you could focus on the dashboard-specific code. Multiple dashboard versions (from the original launch revision through 5960) were cross-referenced to understand how subsystems evolved between builds.

### Phase 5: Deep Binary Analysis

As the reconstructed engine matured, the focus shifted from "can we get this to run" to "are we getting it right." The dashboard looked correct at a glance, but closer inspection kept turning up discrepancies -- colors slightly off, materials rendering flat instead of with transparency falloff, EEPROM settings reading from wrong addresses. The kind of bugs you only notice after staring at the real dashboard for years.

Automated Ghidra scripts against the 5960 XBE exposed the scope of what needed fixing.

#### The FND Table Structure

Every node class exposes functions to the script VM through Function Descriptor (FND) tables. Early RE work found function name strings in the binary and mapped them to node classes. But the tables linking names to code addresses proved elusive -- scanning for 12-byte `{funcPtr, sig, namePtr}` entries returned nothing.

The entries are 24 bytes, not 12. The Xbox compiler used a 16-byte general-purpose member function pointer:

```
+0:  uint32 funcPtr       (4 bytes)
+4:  uint32 thisAdj       (4 bytes, always 0 for single inheritance)
+8:  uint32 vtableIdx     (4 bytes, always 0)
+12: uint32 vbaseOffset   (4 bytes, always 0)
+16: uint16 sig           (2 bytes -- calling convention enum)
+18: uint16 padding       (2 bytes)
+20: uint32 namePtr       (4 bytes -- pointer to UTF-16LE string)
```

On top of that, FND tables live in .bss -- uninitialized data, all zeros on disk. They're populated at runtime by init functions that write entries as `MOV` instruction immediates. No static data to scan.

PRD (Property Descriptor) tables use plain integers, so they ARE statically initialized. A brute-force scan found 35 PRD tables with 299 property labels in one pass. FND tables required parsing the init functions themselves.

#### Extracting Functions from Init Code

The init functions inline constructors directly into struct memory:

```asm
MOV ESI, 0xC0F3FF6B          ; side = D3DCOLOR_RGBA(243, 255, 107, 192)
MOV EDI, 0x0014C000          ; front = D3DCOLOR_RGBA(20, 192, 0, 0)
MOV [EAX+0x04], 0x000294EC   ; name = "FlatSurfaces"
MOV [EAX+0x0C], ESI          ; store side color
MOV [EAX+0x10], EDI          ; store front color
```

Scanning .text for `C7 05` patterns targeting .bss, clustering by proximity, and walking each cluster with register tracking extracted **303 function labels across 25 node classes**. Every script-callable method in the dashboard, with exact code addresses. CConfig had 57. CSavedGameGrid had 36. CDVDPlayer had 35. CLiveAccounts -- the Xbox Live integration Microsoft added between 4920 and 5960 -- had 23 functions that the community's pre-Live codebase never needed.

#### Material Colors

The dashboard's visual identity -- green glow, translucent panels, falloff shading -- comes from ~130 named materials in `Material_Init()`. The community had been building skin files with color values extrapolated from screenshots and hex dumps. Some were close. Some weren't.

The material constructors are inlined -- direct struct writes, no `PUSH`/`CALL`:

```asm
MOV dword ptr [EAX],    0x28BF4     ; vtable = CSolidMatInfo
MOV byte ptr  [EAX+0C], 0xF9        ; R = 249
MOV byte ptr  [EAX+0D], 0x98        ; G = 152
MOV byte ptr  [EAX+0E], 0x19        ; B = 25
MOV byte ptr  [EAX+0F], 0xB2        ; A = 178
```

The vtable address identifies the type: `0x28BF4` = CSolidMatInfo, `0x28BF0` = CFalloffMatInfo. This mattered because several materials had been typed wrong:

| Material | What We Had | What the Binary Says |
|----------|-------------|---------------------|
| OrangeNavType | Solid, alpha 255 | Solid, **alpha 178** |
| MemoryHeaderHilite | Solid (232,199,0) | **Falloff** (199,232,0) / (97,114,0) -- R/G swapped |
| grill grey | Solid (32,32,32) | **Falloff** (32,32,32,0) / (64,64,64,255) |
| grey | Solid (86,100,82) | **Falloff** (71,83,69,0) / (86,100,82,255) |
| dark green panels falloff | Solid (5,35,5) | **Falloff** (0,0,0,0) / (5,35,5,255) |

Three materials typed as solid were actually falloff -- completely different rendering paths. The falloff shader computes viewing-angle transparency; a solid material draws flat. Getting the type wrong loses the X-ray edge glow that defines the dashboard's look.

22 materials in 5960 had no counterpart in 4920 at all -- Xbox Live UI: `LiveChrome`, `OrangeNavType`, `MemoryHeaderHilite`, `footer`, `highlight`, `button`, `image`, `live header`, `orangeEggGlow`, `MotdIcon`.

#### EEPROM Settings

`CConfig` functions read dashboard settings via kernel calls into the Xbox EEPROM. The binary showed that `GetLiveToday` and `GetAcceptedLegalInfo` read from the **same EEPROM location** (`XC_MISC_FLAGS = 0x11`) using different bits:

```
XC_MISC_FLAGS (0x11):
  bit 2: LiveToday disabled (inverted -- 0 means enabled)
  bit 3: Accepted legal info
```

The reconstructed engine had been using the wrong EEPROM type entirely. It worked on modded consoles because nobody checked the real value. With Insignia bringing Xbox Live back, getting these right stopped being academic.

#### Closing the Gaps

Cross-referencing the 303 binary functions against the reconstructed engine found functions that had been rebuilt from User.Interface.X's pre-Live codebase but never properly updated for 5960's Xbox Live additions. The `CSavedGameGrid` in particular had diverged -- Microsoft reorganized the memory manager to show DLC and Xbox Live accounts alongside saved games, partitioning the grid item index space sequentially: saves first, then DLC, then Live accounts. Functions like `IsSavedGameSelected()`, `IsDLContentSelected()`, and `IsXboxLiveAccountSelected()` check which partition the current selection falls in. The CKeyboard gained password masking for Xbox Live passcode entry. CMemoryMonitor added slot counting alongside block counting.

These were all properly implemented from the 5960 disassembly and verified against the behavior described in the XAP scripts that call them.

### A Note on Leaked Code

Parts of Microsoft's Xbox source have circulated in the scene for years. It's no secret. Theseus wasn't built from it -- the engine was reverse-engineered from compiled binaries. Leaked code was useful later for sanity-checking the decompilation and cleaning up ambiguities. But the RE work came first, and the engine compiles and runs independently of any Microsoft source.

## Theseus

Named after the [Ship of Theseus](https://en.wikipedia.org/wiki/Ship_of_Theseus). Start with the retail 5960 XBE. Replace one piece at a time with reconstructed code. At what point is it a different ship?

The original goal wasn't to build a standalone dashboard. It was to understand enough of the XBE to get XIPs and XAPs loading and rendering -- an "I can see this" state where scripts could be tested visually. UIX Lite was working fine as the actual dashboard. The RE work was about understanding the engine so modifications could be made more confidently.

Over time the reconstructed code grew from "enough to load a XIP" to "enough to run the whole dashboard." The core -- lexer, parser, compiler, bytecode VM, scene graph, node reflection, texture loading, mesh rendering -- was reconstructed over approximately six years of iterative work. Each subsystem was disassembled, understood, reimplemented in compilable C++, and tested against real XAP scripts on real hardware. At some point it crossed the line from "a tool for understanding UIX Lite" to "a replacement engine that could stand on its own."

## The Desktop Port

With the engine reconstructed at source level, it could be compiled for any platform. The Xbox dashboard called Direct3D 8 for rendering, Win32 for file I/O, XInput for controllers. Replace those with OpenGL, POSIX, and SDL2, and the same engine runs on a Mac.

That's what UIX Desktop does. D3D8 calls go through a translation layer to OpenGL. `CreateFile`/`ReadFile` become `fopen`/`fread`. XInput becomes SDL2 gamepad input. XBX textures get unswizzled and uploaded as standard OpenGL textures. The XAP scripts don't know they're not on an Xbox.

The desktop port also enabled something unexpected: loading alpha dashboard builds from 2001 recovery discs and running them on modern hardware. The same engine that runs UIX Lite's community XIPs can load Microsoft's pre-retail scripts and render scenes that haven't been seen in this form since the Xbox was in development. Not because nobody had the files -- alpha recovery discs have been floating around for decades -- but because nobody had an engine that could load loose XAP files and pack them into compatible XIPs on the fly.

## TeamUIX

**Original**: acidbath, BobMcGee, Gcue, |ce, JbOnE, MPSnyper, tayior7, slay

**Today**: acidbath, ILTB, Mattie, Milenko, Odb718, ImOkRuOk, headph0ne, BigJx, Rocky5

BigJx handles XIP/XAP script work. Rocky5 does skin presets and XBE patching. The community tests, contributes skins, and keeps the scene alive.
