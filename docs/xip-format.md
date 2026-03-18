# XIP Archive Format

## Overview

XIP is the binary archive format used by the Xbox dashboard to package scripts, textures, meshes, and other assets into a single loadable file. The magic number is `0x30504958` -- `"XIP0"` in ASCII. Microsoft's internal build pipeline compiled XAP scripts, textures, and meshes into XIPs using a packer tool, then optionally signed them with `xipsign`.

The dashboard loads `default.xip` at startup, which contains the root scene and all core UI assets. Additional XIPs (settings, music, memory management, Xbox Live, etc.) are loaded on demand when the user navigates to those sections.

## Archive Structure

The format has three parts: a header, a file table, and a name table that maps names to file entries. The structure was reverse-engineered from the files themselves and later confirmed against header definitions that circulated in the scene.

```
+-------------------+
| XIPHEADER         |  Magic, data start offset, file count, name count, data size
+-------------------+
| FILEDATA[0..N]    |  One per file: data offset, size, type, timestamp
+-------------------+
| FILENAME[0..M]    |  One per name: file index + name string offset
+-------------------+
| Name strings      |  Null-terminated ASCII filenames
+-------------------+
| File data          |  Raw file contents (scripts, textures, meshes, audio)
+-------------------+
```

### XIPHEADER (16 bytes)

```c
struct XIPHEADER {
    DWORD m_dwMagic;      // 0x30504958 ("XIP0")
    DWORD m_dwDataStart;  // Byte offset where file data begins
    WORD  m_wFileCount;   // Number of FILEDATA entries
    WORD  m_wNameCount;   // Number of FILENAME entries
    DWORD m_dwDataSize;   // Total size of all file data
};
```

### FILEDATA (16 bytes each)

```c
struct FILEDATA {
    DWORD m_dwDataOffset;  // Offset from data start to this file's content
    DWORD m_dwSize;        // Size in bytes
    DWORD m_dwType;        // File type (see below)
    DWORD m_dwTimestamp;   // Build timestamp
};
```

### FILENAME (4 bytes each)

```c
struct FILENAME {
    WORD m_wFileDataIndex;  // Index into the FILEDATA array
    WORD m_wNameOffset;     // Offset into the name string table
};
```

The name/file split is worth noting: multiple names can point to the same file data entry. This was used for mesh references -- a `.meta` file could alias another mesh without duplicating the vertex data.

### File Types

Microsoft defined seven types in `xip.h`:

| Value | Name | Content |
|-------|------|---------|
| 0 | `XIP_TYPE_GENERIC` | XAP scripts, audio, and anything else |
| 1 | `XIP_TYPE_MESH` | XM mesh files (vertex + index buffers) |
| 2 | `XIP_TYPE_TEXTURE` | XBX texture files |
| 3 | `XIP_TYPE_WAVE` | WAV audio (often IMA ADPCM compressed) |
| 4 | `XIP_TYPE_MESH_REFERENCE` | Points to another mesh by name |
| 5 | `XIP_TYPE_INDEXBUFFER` | Raw index buffer (part of a mesh) |
| 6 | `XIP_TYPE_VERTEXBUFFER` | Raw vertex buffer (part of a mesh) |

Types 5 and 6 are used internally by the packer -- during XIP creation, meshes are split into separate vertex buffer and index buffer entries. When the XIP is loaded, the mesh reader reassembles them into a complete mesh object.

There is no compression. Files are stored raw. This made sense on the Xbox where CPU time was more valuable than disk space, and the hard drive was fast enough for sequential reads.

## XIP Signatures

Microsoft's `xipsign` tool (found at `xbox/ui/private/xipsign/`) computed HMAC signatures over 64KB chunks of each XIP file. The signing was straightforward: read the XIP in 64KB blocks, compute a signature per block using Xbox crypto APIs (`xcrypt.h`), write the signatures to a protection file.

Every dashboard version checked these signatures before loading XIPs. When the modding community started patching out the check, Microsoft responded by moving the signature verification code to different locations in the XBE with each dashboard update. The community would find the new location, NOP out the branch, and the cycle would repeat. Once you could run unsigned XBEs (via modchip or softmod), the XIP signatures were just a single branch instruction to patch out -- wherever Microsoft hid it.

### How the XIP Loader Works

The XIP loading function (`CFileBuffer::SetFile`, at `0x0003DF54` in the 5960 binary) validates and parses the archive in a specific sequence. The first thing it checks is the magic number:

```asm
0003DF6C: PUSH ESI                          ; header buffer
0003DF6D: CALL 0x0003D710                   ; ReadFile (16 bytes = XIP header)
0003DF72: TEST EAX, EAX
0003DF74: JZ   0x0003DFF3                   ; read failed -> bail
0003DF76: CMP  dword ptr [ESI], 0x30504958  ; compare against "XIP0" magic
0003DF7C: JNZ  0x0003DFF3                   ; not XIP0 -> bail
```

If the magic passes, it reads the three directory sections using counts and sizes from the header:

```asm
; Read FILEDATA entries (header+0x10 = count, each entry 16 bytes)
0003DF7E: MOVZX EDX, word ptr [EDI + 0x10]  ; FILEDATA count
0003DF82: SHL   EDX, 0x4                     ; count * 16 = total bytes
0003DF85: PUSH  EDX
0003DF86: CALL  0x00072D2C                   ; malloc(count * 16)

; Read FILENAME entries (header+0x12 = count, each entry 4 bytes)
0003DFA3: MOVZX EDX, word ptr [EDI + 0x12]  ; FILENAME count
0003DFA7: SHL   EDX, 0x2                     ; count * 4 = total bytes

; Read name strings block (remaining bytes after directories)
0003DFC8: MOVZX EDX, word ptr [EDI + 0x10]  ; FILEDATA count
0003DFCC: MOVZX EAX, word ptr [EDI + 0x12]  ; FILENAME count
0003DFD0: MOV   ESI, [EDI + 0x0C]           ; total file size
0003DFD3: LEA   ECX, [EAX + EDX*4 + 4]      ; header + directories size
0003DFD7: SHL   ECX, 0x2                     ; in bytes
0003DFDA: SUB   ESI, ECX                     ; remaining = name strings
```

File lookups use `bsearch` on the sorted FILENAME directory:

```asm
; FindFileInXIP -- binary search on FILENAME entries
0003E2D2: PUSH  0x3E270                      ; compare function (string compare)
0003E2D7: PUSH  0x4                          ; element size (4 bytes per FILENAME entry)
0003E2E4: PUSH  EDX                          ; FILENAME count
0003E2E9: PUSH  EAX                          ; FILENAME array base
0003E2EE: PUSH  ECX                          ; search key (filename string)
0003E2EF: CALL  0x00074040                   ; bsearch()
```

This is why XIP tools must sort the FILENAME directory alphabetically -- the engine uses `bsearch()`, which requires sorted input. An unsorted directory produces lookup failures and missing assets.

## File Formats Inside a XIP

### XAP Scripts (.xap) -- Type 0

Plain text files in VRML97 syntax with JavaScript-like inline scripts. The dashboard compiles them at load time (not pre-compiled to bytecode on disk), which is what made script modding so accessible -- edit the text, reload, see changes. See [XAP Script System](xap-scripting.md) for the full format.

### XBX Textures (.xbx) -- Type 2

Xbox-native texture files using the XPR0 (Xbox Packed Resource) format:

```
+------------------+
| XPR_HEADER       |  Magic (0x30525058 "XPR0"), total size, header size
+------------------+
| D3D Resource     |  Pixel format, dimensions, mipmap info (encoded in GPU register format)
+------------------+
| Pixel Data       |  DXT-compressed, swizzled (Morton-order) pixel data
+------------------+
```

Supported pixel formats: DXT1 (4:1 compression, 1-bit alpha), DXT3 (explicit 4-bit alpha), DXT5 (interpolated alpha), plus uncompressed formats like A8R8G8B8, R5G6B5, L8, A8, and palettized P8.

The "swizzling" is a Morton Z-order curve -- pixels are laid out in a GPU-friendly pattern rather than linear scanline order. The desktop port deswizzles the data in software before uploading to OpenGL. Microsoft's format packed the dimensions and format info directly into NV2A GPU register fields, so parsing these textures required understanding the GPU's register layout. But the format itself was consistent and well-structured -- once you understood the register encoding, every XBX file parsed the same way.

Microsoft also reused the `.xbx` extension for non-texture data. `TitleMeta.xbx` and `SaveMeta.xbx` in the UDATA directory were plain UTF-16LE text files with save game metadata. These weren't in XPR0 format at all -- just text with a misleading extension. And none of them were signed.

### XM Meshes (.xm) -- Type 1

3D geometry files exported from 3ds Max via Microsoft's `wrl2xm` converter. The tool read VRML97 `.wrl` files and compiled them into Xbox-native mesh data:

```
+--------------------+
| Mesh Header        |  FVF code, vertex count, index count, primitive type, material name
+--------------------+
| Vertex Buffer      |  Array of vertices (layout defined by FVF)
+--------------------+
| Index Buffer       |  16-bit triangle indices
+--------------------+
```

The FVF (Flexible Vertex Format) code describes each vertex's layout:
- `D3DFVF_XYZ` (0x002): 3D position (3 floats, 12 bytes)
- `D3DFVF_NORMAL` (0x010): Surface normal (3 floats, 12 bytes)
- `D3DFVF_NORMPACKED3`: Xbox-specific packed normal (1 DWORD encoding 3 x 11-bit components)
- `D3DFVF_DIFFUSE` (0x040): Vertex color (DWORD, ARGB)
- `D3DFVF_TEX1` (0x100): Texture coordinates (2 floats, 8 bytes)

Materials are referenced by name string. When a mesh says its material is `"chrome_blue"`, the material system looks up that name in the loaded `MaxMaterial` nodes.

### XTF Fonts (.xtf)

Xbox bitmap fonts with glyph tables, character maps, and kerning data. The format stores glyph metrics (advance width, bearing, bounding box) and bitmap data for each character. On Xbox, glyphs were rendered directly from the font file. On desktop, the glyph data is rasterized to OpenGL textures.

### Audio (.wav) -- Type 3

WAV files, often using Xbox IMA ADPCM compression (format tag `0x0069`). The desktop port includes a software IMA ADPCM decoder that converts these to PCM at load time for playback through SDL_mixer.

## Loading Process

When the dashboard starts:

1. **Load XIP**: Read the XIPHEADER, file table, and name table into memory
2. **Find default.xap**: Search the name table for `"default.xap"`
3. **Parse**: The lexer/parser/compiler process the XAP text into a scene graph with executable bytecode
4. **Resolve assets**: When the scene graph references a texture or mesh by name, `FindObjectInXIP()` locates it in the archive
5. **Skin override**: Before checking the XIP, the system checks `Q:\Skins\{active_skin}\` for an override file with the same name. If found, the skin's version is used instead.

## Sub-XIPs and Lazy Loading

The dashboard uses ~28 XIP files. Only `default.xip` is loaded at boot. The rest are loaded on demand when the user navigates to that section -- triggered by `Inline` nodes in the scene graph that reference sub-XAP files in other XIPs.

```
Q:\Xips\
  default.xip      Root scene, main menu, core UI
  mainmenu5.xip    Main menu panels
  settings3.xip    Settings panels
  music2.xip       Music player / soundtrack manager
  memory2.xip      Memory management
  dvd.xip          DVD player UI
  keyboard.xip     On-screen keyboard
  ...
```

On the desktop, XIPs can be loaded in two modes:
- **Compiled mode**: Load the `.xip` binary directly (same as Xbox)
- **Extracted mode**: Load from a directory of extracted files (for development -- edit `.xap` scripts and see changes without rebuilding the XIP)

## XIP Tool

The `tools/xiptool.c` utility extracts and creates XIP archives. It builds on 20 years of community XIP tooling -- Voltaic's **pixit**, MaTiAz5's **UnXiP**, and VulgasProfanum's contributions, among others:

```bash
# Extract a XIP
xiptool -x default.xip output_dir/

# Create a XIP from a directory
xiptool -c output_dir/ new_default.xip
```

When extracting, the tool reassembles mesh entries (recombining split vertex/index buffer entries into complete `.xm` files with their original MESHFILEHEADER). When creating, it splits `.xm` files back into IB+VB entries and auto-generates mesh reference entries so scripts can find meshes by name.

A critical detail: the FILENAME directory must be **alphabetically sorted** (case-insensitive). The engine uses `bsearch()` to look up files by name, which requires sorted input. Early versions of community XIP tools didn't sort the directory, producing XIPs that looked valid but couldn't be loaded -- the binary search would miss entries that were out of order. Microsoft's original `xip.exe` build tool (found on a 3729 recovery disc in `mkxips.cmd`) handled this internally.

The FILEDATA array (which stores offsets, sizes, and types) does NOT need to be sorted -- it's accessed by index, not by name. The FILENAME directory is a sorted index into the unsorted FILEDATA array. This separation means you can pack data in any order (IB first, VB second, generic files last) while maintaining fast name lookups.
