# Xbox Live and the 5960 Problem

## Why 4920?

For fifteen years, the Xbox modding community built on dashboard version 4920. There were good reasons. It was stable, well-understood, and the community had mapped enough of the binary to hex-patch colors, paths, and signatures reliably. JbOnE's UIX, the tHc patches, BSX -- everything targeted 4920 or its close relatives.

4920 had Xbox Live support. It shipped after Live launched -- account management, friend lists, the works. But the community never used any of it. Modded consoles stayed off Xbox Live by convention, and Microsoft kept pushing dashboard updates with improved Live features. Builds 5659 and 5960 added better DLC management, refined account selection, Live Today notifications, and more polished UI. The community watched the "Live 2.0" dashboards come and go, and stayed on 4920 because it worked and they didn't need any of the new stuff.

Then Microsoft shut down Xbox Live on April 15, 2010, and the question became permanently irrelevant. Nobody was building dashboard mods that needed Live support. The community's 4920 base was fine.

## Insignia Changes Everything

In 2023, [Project Insignia](https://insignia.live/) did something nobody expected: they brought Xbox Live back.

The technical approach was elegantly simple. Original Xbox Live was a client-server system. The Xbox connected to Microsoft's authentication servers, verified credentials, and got access to multiplayer, friends lists, content downloads, and messaging. Insignia rebuilt the server side. Point your Xbox's DNS at Insignia's servers, and the console's XOnline APIs connect to Insignia thinking it's Microsoft. The Xbox has no idea. Every `XOnlineLogon()`, every `XOnlineGetNotification()`, every friend request and game invite flows through the original APIs to Insignia's replacement infrastructure.

This changed the calculus for dashboard modding overnight.

UIX Lite on 4920 had Xbox Live code, but it was the earlier implementation. Missing DLC content management, missing the refined account selection flow, missing Live Today notifications. The community's custom XIPs had replaced most of the stock scripts anyway -- and nobody had tested whether 4920's Live code still worked correctly through custom XAPs, because there was no Live service to test against.

Dashboard 5960 was different. It was the final retail build -- the most complete and most tested Xbox Live implementation Microsoft shipped. It was also the version Insignia targeted for compatibility testing. If you wanted a modded dashboard that worked reliably with Insignia, 5960 was the safe bet.

So that's what we did.

## What Changed Between 4920 and 5960

The community had treated 4920 and 5960 as "basically the same dashboard with minor updates." The actual diff was bigger than anyone expected. What follows is documented from the 5960 XBE binary directly -- every address, every byte, verifiable by anyone with Ghidra and a retail `xboxdash.xbe`.

### The LiveAccounts Node

The `LiveAccounts` node existed in 4920, but Microsoft expanded it significantly for 5960. The final version has 23 script-callable functions -- a complete account management system. The FND (Function Descriptor) table is populated at runtime by an init function. The entries are 24 bytes each -- 16 bytes for the member function pointer (MSVC general representation), 2 bytes for the calling convention, 2 bytes padding, 4 bytes for the name string pointer.

Here's what the init function writes for a few key entries:

```
CLiveAccounts FND init (inside cluster at 0x000C9CC4-0x000CE4DE):

  GetAccountName     @ 0x00069750  sig=4 (string,int)
  GetNumAccountsOnHD @ 0x000696F0  sig=2 (int,void)
  Logon              @ 0x000699E0  sig=28 (void,int,int,string)
  Logoff             @ 0x00069AE0  sig=1 (void,void)
  GetResult          @ 0x000697F0  sig=8 (string,void)
  IsPasswordEnabled  @ 0x000698B0  sig=3 (int,int)
  PersistUser        @ 0x00069E60  sig=14 (void,int)
  LaunchEntryPoint   @ 0x00069E90  sig=26 (void,int,int,string)
  GetMessageOfTheDay @ 0x0006A330  sig=8 (string,void)
  IsVoiceAllowed     @ 0x0006A120  sig=2 (int,void)
```

Some of these are remarkably simple. `Refresh()` at `0x000696E0` is two instructions:

```asm
000696E0: MOV byte ptr [ECX + 0x1c], 0x1    ; set m_bNeedRefresh = true
000696E4: RET
```

It sets a dirty flag. The actual re-enumeration happens in the node's `Advance()` function on the next frame. `ClearLastLogonUser()` at `0x0006A0C0` is similarly minimal:

```asm
0006A0C0: MOV byte ptr [0x0017F42C], 0x0    ; clear global "has last logon" flag
0006A0C7: RET
```

Others are substantial. `Logoff()` at `0x00069AE0` tears down the XOnline session, clears the task handle, resets the account index, and zeros out the friend/invite state:

```asm
00069AE0: SUB ESP, 0xC
00069AEB: MOV ESI, ECX                      ; save 'this'
00069AF1: CALL 0x00069DD0                   ; internal cleanup
00069AF8: CALL 0x0006A290                   ; clear notification state
00069AFD: MOV EAX, [ESI + 0x40]            ; m_hLogonTask
00069B02: CMP EAX, EBX                     ; NULL check
00069B04: JZ  0x00069B50                    ; skip if no active session
    ...
00069B35: CALL 0x0011680B                   ; XOnlineTaskClose(hTask)
00069B3A: MOV [ESI + 0x40], EBX            ; m_hLogonTask = NULL
00069B3D: MOV [ESI + 0x24], 0xFFFFFFFF     ; m_nCurrentAccount = -1
00069B44: MOV [ESI + 0x1D], BL             ; m_bLoggedOn = false
00069B47: MOV [ESI + 0x30], EBX            ; m_nGameInvites = 0
00069B4A: MOV [ESI + 0x34], EBX            ; m_nFriendInvites = 0
00069B4D: MOV [ESI + 0x38], EBX            ; m_nFriendsOnline = 0
```

`PersistUser()` at `0x00069E60` stores which account to auto-select on next boot:

```asm
00069E60: MOV EAX, [ESP + 0x4]             ; index parameter
00069E64: TEST EAX, EAX
00069E66: JL  skip                          ; reject negative index
00069E68: CMP EAX, [ECX + 0x20]            ; compare with m_nNumAccounts
00069E6B: JG  skip                          ; reject out-of-range
00069E6D: MOV [ECX + 0x28], EAX            ; m_nPersistedUser = index
00069E70: RET 4
```

`IsVoiceAllowed()` at `0x0006A120` is a single member read:

```asm
0006A120: MOV DL, byte ptr [ECX + 0x1E]    ; m_bVoiceAllowed
0006A123: XOR EAX, EAX
0006A125: TEST DL, DL
0006A127: SETNZ AL                          ; return (m_bVoiceAllowed != 0)
0006A12A: RET
```

### The Account Structure

Xbox Live accounts are stored on the hard drive starting at sector `0x1300`. Each account is a `0x6C`-byte structure:

```
Offset  Size  Field
0x00    8     XUID (64-bit user ID)
0x08    4     Reserved (must be 0)
0x0C    16    Gamertag (ASCII, null-terminated)
0x1C    4     Flags (bit 0 = password enabled)
0x20    4     Passcode (4 bytes)
0x24    20    Domain (ASCII, null-terminated)
0x38    24    Realm (ASCII, null-terminated)
0x50    20    Confounder (encrypted with 3DES)
0x64    8     Verification (HMAC-SHA1 digest)
```

The confounder at offset `0x50` is encrypted using Triple DES with a key derived from the Xbox's unique HD key (stored in the EEPROM). The key derivation uses two HMAC rounds with fixed seed keys:

```
seed_key_a = 2B B8 D9 EF D2 04 6D 9D 1F 39 B1 5B 46 58 01 D7
seed_key_b = 1E 05 D7 3A A4 20 6A 7B A0 5B CD DF AD 26 D3 DE
IV         = 7B 35 A8 B7 27 ED 43 7A

tempA = HMAC-SHA1(seed_key_a, XboxHDKey)
tempB = HMAC-SHA1(seed_key_b, XboxHDKey)
3DES_key = tempA[0..3] + tempB[0..19]  (24 bytes)
```

Profile verification computes HMAC-SHA1 over bytes `0x00` through `0x63` using the HD key, then compares the first 8 bytes of the digest against the verification tag at `0x64`. If they match, the account is valid for this specific Xbox. Accounts are hardware-bound -- you can't copy them between consoles without re-signing.

The `IsPasswordEnabled()` function at `0x000698B0` reads bit 0 of the flags field. The struct offset math is visible in the disassembly:

```asm
000698C9: IMUL EAX, EAX, 0x70              ; account_index * sizeof(XONLINE_USER)
000698CC: MOV EAX, [EAX + ESI + 0x60]      ; flags at offset 0x60 in XONLINE_USER
000698D0: AND EAX, 0x1                      ; bit 0 = password enabled
```

(The XDK's `XONLINE_USER` struct is larger than the raw disk format -- 0x70 bytes including runtime state.)

### SavedGameGrid Expansion

The memory manager went from showing saved games and soundtracks to showing saved games, soundtracks, downloadable content, AND Xbox Live accounts -- all in the same grid. Microsoft partitioned the grid item index space:

```
Grid items: [saves][DLC][Live accounts]

Index 0..N-1         = saved games
Index N..N+D-1       = downloadable content
Index N+D..N+D+A-1   = Xbox Live accounts
```

The selection query functions check which partition the current index falls in. `IsSavedGameSelected()` at `0x0004B830`:

```asm
0004B830: MOV ESI, ECX                      ; save 'this'
0004B833: MOV EAX, [ESI + 0xB0]            ; m_curDevUnit
0004B839: MOV ECX, [ESI + 0x8C]            ; m_curTitle
0004B83F: IMUL EAX, EAX, 0x944             ; device * sizeof(TitleEnum)
0004B845: PUSH 0x0                          ; type = SAVE
0004B847: ADD EAX, 0x182088                 ; base of title enum array
0004B84C: PUSH EAX
0004B84D: CALL 0x00042740                   ; GetContentCount(titleData, type)
0004B852: MOV EDX, [ESI + 0x94]            ; m_curGridItem
0004B85A: CMP EDX, EAX                     ; curGridItem < saveCount?
0004B85C: SETL CL                           ; result = (curGridItem < saveCount)
0004B860: MOV EAX, ECX
0004B862: RET
```

`IsGameTitleSelected()` at `0x0004BAF0` checks that the current title isn't a soundtrack entry or a Live accounts entry:

```asm
0004BAF0: MOV EAX, [ECX + 0x8C]            ; m_curTitle
0004BAF8: TEST EAX, EAX
0004BAFA: JL  return_false                  ; curTitle < 0 = nothing selected
0004BAFC: CMP EAX, [ECX + 0x100]           ; m_nSoundtrackTitle
0004BB02: JZ  return_false                  ; it's the soundtrack entry
0004BB04: CMP EAX, [ECX + 0x108]           ; m_nXboxLiveAccountTitle
0004BB0A: JZ  return_false                  ; it's the Live accounts entry
0004BB0C: MOV AL, 0x1                       ; it's a game title
0004BB11: RET
```

The field at offset `0x108` (`m_nXboxLiveAccountTitle`) is the 5960 addition -- an index in the title list where Live accounts appear, analogous to `m_nSoundtrackTitle` at `0x100`. Both are `-1` when their content type isn't present on the current device.

### New XAP Scripts

The 5960 `default.xap` gained 974 lines over 4920. The additions are almost entirely Xbox Live integration:

```javascript
// 5960 default.xap -- Xbox Live account manager node (expanded from 4920)
DEF theLiveAccounts LiveAccounts
{
    bLogon        FALSE
    currentAccount -1
    newAccountFromMU 0
}

// Navigate to Xbox Live dashboard entry point
function GoToXoDashEntryPoint(ActiveControllerPort, bClearPasscode, EntryPoint)
{
    theLiveAccounts.LaunchEntryPoint(ActiveControllerPort, bClearPasscode, EntryPoint);
}

// PS Bug #33608: disconnect from Live when entering CD player
// (prevents notification sounds during music playback)
function LogOff()
{
    theLiveAccounts.Logoff();
}
```

The `PS Bug` references are ProductStudio, Microsoft's internal bug tracker. Bug #33608 ("You are still seen as Online when playing an Audio CD") and #33611 ("You can hear Live Now notification sounds while playing Audio CD") are visible in the extracted XAP scripts from the retail XIPs. Microsoft's QA process, frozen in the scripts they shipped to every retail console.

Four new XIPs appeared in 5960 that don't exist in 4920:

| XIP | Purpose | Notes |
|-----|---------|-------|
| `AccountSelection.xip` | Xbox Live account picker UI | Shows gamertags, handles MU account import |
| `LiveToday.xip` | Live Today screen | News, friend status, game invites |
| `PasscodeVerify.xip` | Parental control passcode entry | Uses CKeyboard asterisk formatting mode |
| `WaitCursor.xip` | Loading spinner | Displayed during async Live operations |

### New Materials

22 materials were added for Xbox Live UI elements. The material setup code in `Material_Init()` (code region `0x00057CDE` - `0x0005A1FB`) creates each material by inlining the constructor -- no function calls, just direct struct writes. The vtable pointer identifies the type:

```asm
; LiveChrome -- the purple accent color for Xbox Live panels
; vtable 0x28BEC = Xbox Live solid material variant
00059E24: MOV dword ptr [EAX + 0x04], 0x28CF8   ; name -> "LiveChrome"
00059E2B: MOV dword ptr [EAX + 0x08], ESI        ; flags
00059E2E: MOV dword ptr [EAX],       0x28BEC     ; vtable
00059E34: MOV byte ptr  [EAX + 0x0C], 0x9D       ; R = 157
00059E38: MOV byte ptr  [EAX + 0x0D], 0x6D       ; G = 109
00059E3C: MOV byte ptr  [EAX + 0x0E], 0xC2       ; B = 194
00059E40: MOV byte ptr  [EAX + 0x0F], 0xFF       ; A = 255
```

The same purple `(157, 109, 194)` appears on `footer`, `highlight`, `button`, and `image` -- a consistent Xbox Live accent color. Other additions include `OrangeNavType` (249, 152, 25, 178) for Live navigation text and `orangeEggGlow` for notification pulses.

The methodology for extracting material colors from the binary -- vtable identification, inlined constructor patterns, the corrections to community-guessed values -- is detailed in the [reverse engineering](reverse-engineering.md) section.

### EEPROM Changes

Two Live-related settings landed in `XC_MISC_FLAGS` (EEPROM type `0x11`). The disassembly of `GetLiveToday()` at `0x0003AD80` shows the bit extraction:

```asm
0003AD80: SUB ESP, 0x8
0003AD83: PUSH 0x0                          ; ResultLength (out)
0003AD85: PUSH 0x4                          ; ValueBufferLength
0003AD87: LEA EAX, [ESP + 0x8]
0003AD8B: PUSH EAX                          ; ValueBuffer
0003AD8C: LEA ECX, [ESP + 0x10]
0003AD90: PUSH ECX                          ; Type (out)
0003AD91: PUSH 0x11                         ; XC_MISC_FLAGS
0003AD93: CALL 0x0006C38E                   ; ExQueryNonVolatileSetting
0003AD98: MOV EAX, [ESP]                    ; read the value
0003AD9B: SHR EAX, 0x2                      ; shift right 2 (isolate bit 2)
0003AD9E: NOT EAX                           ; invert (bit 2 = "disabled")
0003ADA0: AND EAX, 0x1                      ; mask to single bit
0003ADA3: ADD ESP, 0x8
0003ADA6: RET
```

`GetAcceptedLegalInfo()` at `0x0003AE00` uses the same EEPROM location but bit 3:

```asm
0003AE18: MOV EAX, [ESP]                    ; read XC_MISC_FLAGS value
0003AE1B: SHR EAX, 0x3                      ; shift right 3 (isolate bit 3)
0003AE1E: AND EAX, 0x1                      ; mask to single bit
0003AE21: ADD ESP, 0x8
0003AE24: RET
```

Two functions, same EEPROM read, different `SHR` operand. The `SetLiveToday()` counterpart at `0x0003ADB0` does the inverse -- reads the flags, clears or sets bit 2 with `AND ~4` / `OR 4`, writes back with `ExSaveNonVolatileSetting`.

The previous community implementation had been using EEPROM type `0x08` (parental controls) with entirely different bit positions. It worked on modded consoles because no software was checking the real flags. With Insignia bringing Xbox Live back, the real values matter.

## The Implementation

The XOnline APIs are the backbone. On the original Xbox, these linked against `xonline.lib` and talked to Microsoft's servers. With Insignia, they talk to Insignia's servers. The API surface is identical. The Xbox doesn't know the difference.

```cpp
// This code works with both Microsoft's dead servers and Insignia's live ones.
// The Xbox resolves "xonline.xbox.com" via DNS -- Insignia's DNS returns
// Insignia's IP. The XOnline APIs handle everything else.

HRESULT hr = XOnlineStartup(NULL);
hr = _XOnlineGetUsersFromHD(g_Users, &g_NumUsers);
hr = XOnlineLogon(&g_Users[index], NULL, 0, NULL, &hLogonTask);

while (XOnlineTaskContinue(hLogonTask) == XONLINETASK_S_RUNNING)
    Sleep(50);

XOnlineTaskClose(hLogonTask);
```

The reconstructed engine loads accounts from disk, verifies their signatures against the HD key, displays gamertags through the script system's text rendering, handles password entry through the keyboard node's asterisk formatting mode, and manages the logon flow through XOnline's async task system.

The `SavedGameGrid` integration is driven by a single global: `g_NumUsers`. When `CLiveAccounts` loads accounts via `_XOnlineGetUsersFromHD()`, the count feeds into `GetXboxLiveAccountsCount()`. The grid adds them after saves and DLC in the index space. The XAP scripts query `IsXboxLiveAccountSelected()` to know when to show the account management panel instead of the save details panel.

### What Works

- **Account enumeration**: `_XOnlineGetUsersFromHD()` loads all stored accounts
- **Account display**: Gamertags render in the Live panels
- **Logon flow**: `XOnlineLogon()` authenticates with Insignia
- **Profile verification**: 3DES/HMAC signature validation against HD key
- **Friend/invite queries**: `GetGameInvites()`, `GetFriendInvites()`, `GetNumberOfFriendsOnline()`
- **Grid integration**: SavedGameGrid shows Live accounts alongside saves and DLC
- **EEPROM settings**: LiveToday and AcceptedLegalInfo read the correct bits from `XC_MISC_FLAGS`
- **Selection queries**: IsSavedGameSelected/IsDLContentSelected/IsXboxLiveAccountSelected partition correctly

### What's Still In Progress

- **Message of the Day**: The MOTD download uses private XOnline APIs that aren't in the standard XDK headers. The function exists in the binary at `0x0006A330` but the internal API signatures need to be recovered from the library exports.
- **Account copy to MU**: `StartXboxLiveAccountCopy()` validates the selection but the MU write path isn't wired up yet.
- **ShowIcon**: The Live notification icon loading path (`0x0006A530`) reads an icon path from offset `0x1078C` in the CLiveAccounts object, loads the texture, and sets a global pointer. The path is understood but not connected to the texture pipeline.

## The Easter Egg

While mapping CConfig's 57 functions, one stood out: `ToggleNoisyCamera` at `0x0003B9E0`.

```asm
0003B9E0: PUSH EBX
0003B9E1: PUSH 0x1FFC4                      ; path -> L"T:\\NoisyCamera"
0003B9E6: CALL 0x0006C63A                   ; NtFileExists
0003B9EB: CMP EAX, -0x1
0003B9EE: SETNZ BL                          ; BL = file exists
0003B9F3: JZ  0x0003BA0B                    ; if doesn't exist, skip delete
0003B9F5: PUSH 0x1FFC4
0003B9FA: CALL 0x0006C707                   ; DeleteFile("T:\\NoisyCamera")
0003B9FF: TEST BL, BL
0003BA01: SETZ AL                           ; AL = !exists (toggled)
0003BA04: MOV [0x0017C614], AL              ; g_bNoisyCamera = toggled state
0003BA09: POP EBX
0003BA0A: RET
```

If `T:\NoisyCamera` exists, delete it and disable the effect. If it doesn't exist, the file gets created (by the caller) and the effect enables. The file's mere presence on the T: partition controls a camera shake effect in the dashboard's 3D viewport.

The trigger? In `default.xap`, the Joystick node has a `secretKey` property set to `"YX"`. Press Y then X on the controller, and `CJoystick::ProcessSecretKeySequence` fires `theConfig.ToggleNoisyCamera()`. A hidden camera shake mode, toggled by a secret button combo, persisted as an empty file on the T: partition. Microsoft shipped an easter egg in every retail dashboard from 5960 onward.

## The Circle

The community spent fifteen years on 4920 because Xbox Live was dead and nobody needed it. Then Insignia brought Live back, and suddenly the community needed to move to 5960 -- the build they'd been ignoring because it was "too complex" with all the Live integration.

The complexity that made 5960 unattractive for modding in 2005 is exactly what makes it necessary now. The 23 CLiveAccounts functions, the 8 SavedGameGrid extensions, the 22 new materials, the 974 lines of Live-aware scripts -- all of that is the cost of entry for a dashboard mod that works with Insignia.

UIX Lite runs on 5960. The reconstructed engine implements the Live functions from the binary. The XOnline APIs connect to Insignia. Twenty years after Microsoft built the original Xbox Live dashboard, a community-rebuilt engine is running community scripts on a community-rebuilt Live service, on the hardware Microsoft designed, using the APIs Microsoft wrote.

The ship has been rebuilt plank by plank, and it's sailing somewhere Microsoft never intended.
