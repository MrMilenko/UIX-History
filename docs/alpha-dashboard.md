# Xbox Dashboard Evolution: From Flat Menus to the Final XIP

Every Xbox dashboard build from December 2000 through the final retail update, traced through actual recovery disc images and XDK packages. This is what the dashboard looked like at each stage -- what Microsoft added, what they cut, and what they left behind as dead code.

These alpha builds can be loaded in the [UIX Desktop](https://github.com/OfficialTeamUIX/UIX-Desktop) engine today. The screenshots below are from 2026, running 25-year-old dashboard code on macOS.

![Build 3424 MainMenu2 -- the 4-pod menu with GAMES](images/alpha-running/3424-mainmenu2-4pods.png)
*Build 3424 (April 2001) running in UIX Desktop. Four pods: Memory, Music, Games, Settings. Games is selected by default. This menu was cut to three pods by May 2001.*

---

## Build 3146 -- December 2000 (Alpha I)

The earliest known dashboard. No 3D graphics. No XAP scripts. Just flat `.mnu` text files and BMP buttons on a 2D screen.

The `.mnu` format is a simple scene layout language: each line specifies a type (`t` for text, `h` for header, `i` for image), position coordinates, colors, and a label:

```
h 134.0   9.0 0xFFDFF77F 0xFFFFFFFF Display_Settings
h 129.0  83.0 0xFFDFF77F 0xFFFFFFFF Network_Settings
t 126.0  14.0 0xFF57932E 0xFFFFFFFF Video_:
```

Menus for system info, memory, video resolution, EEPROM access, network settings, timezone. This was a devkit setup utility, not a consumer UI. No games, no music player, no 3D.

## Build 3223 -- February 2001 (Alpha II)

Still `.mnu` flat menus. Added memory unit formatting (`muformat.mnu`, `muname.mnu`). The first signs of removable storage support. Still no XAP engine.

## Build 3424 -- April 2001 (Alpha II)

Everything changed. The full 3D XAP dashboard appeared -- VRML97 scene graphs, JavaScript scripting, Direct3D 8 rendering. No XIP archives. Everything is loose files in directories:

```
TDATA/fffe0000/
    default.xap              -- root scene (391 lines)
    games.xap                -- game launcher bridge (766 lines)
    music.xap                -- music menu (987 lines)
    memory.xap               -- memory manager (590 lines)
    settings.xap             -- settings (1,388 lines)
    GameHilite_01.bmp        -- selection highlight texture

    MainMenu/                -- original main menu (926 lines)
    MainMenu2/               -- redesigned main menu (2,743 lines)

    Games/                   -- game launcher tunnel mesh
    NewGames/                -- game list browser (1,547 lines + meshes)
    Games_Title/             -- game detail screen (3,303 lines + meshes)

    Memory/                  -- memory manager UI
    Memory_Files/            -- save game browser
    Music/                   -- music player (3MB default.xap!)
    Music_Play/              -- music playback UI

    NewSettings/             -- redesigned settings
    Settings_Audio/          -- audio settings
    Settings_Clock/          -- clock settings
    Settings_Language/       -- language settings
    Settings_Video/          -- video settings
```

Two versions of the main menu exist side by side (`MainMenu/` and `MainMenu2/`), suggesting active iteration. The retail dashboard shipped with `MainMenu5`.

### Four Equal Menus

The root `default.xap` defines four top-level menu items, loaded the same way:

```javascript
DEF theGamesInline Inline {
    visible false
    url "Games.xap"
    function onLoad() {
        theGamesInline.children[0].theGamesMenu.GoTo();
    }
}

DEF theMusicInline Inline { visible false; url "Music.xap" ... }
DEF theMemoryInline Inline { visible false; url "Memory.xap" ... }
DEF theSettingsInline Inline { visible false; url "Settings.xap" ... }
```

Games, Music, Memory, Settings -- all equal, all loaded the same way. And the default selection:

```javascript
nCurMainMenuItem = 2; // start with games selected
```

**Games was the default.** Memory (0), Music (1), Games (2), Settings (3). When you booted the alpha dashboard, the cursor started on Games.

### theTitleCollection: The First-Class Game API

The root script declares a `TitleCollection` node at the top level:

```javascript
DEF theTitleCollection TitleCollection
{
    function OnDiscRequest()
    {
        log("Please insert the disc...");
        // TODO: A real prompt that the user can see!
    }
}
```

This node had direct API methods the scripts could call:
- `theTitleCollection.GetTitleCount()` -- how many games on disk
- `theTitleCollection.GetTitleName(index)` -- game name
- `theTitleCollection.GetTitleID(index)` -- title ID
- `theTitleCollection.LaunchByID(titleID, "")` -- launch a game
- `theTitleCollection.SetLanguage(lang)` -- localization

One line to launch a game: `theTitleCollection.LaunchByID(curTitleID, "")`. The community spent 20 years rebuilding this with music player hijacks, kernel device path construction, and Settings INI file workarounds.

### The Game Browser

![Build 3424 game launcher running in UIX Desktop](images/alpha-running/3424-game-launcher.png)
*The "GAMES" browser from build 3424. Eight-item scrollable list, selection highlight, info panel on the right. The list is empty because the `TitleCollection` API doesn't exist in the desktop engine -- this is where "Go buy some games!" would appear.*

`NewGames/default.xap` (1,547 lines) provided an 8-item scrollable list with dedicated 3D meshes:

```javascript
function GetTitleList()
{
    var titleCount = theTitleCollection.GetTitleCount();
    var str = titleCount + " ";
    if (titleCount == 1)
        str = str + theTranslator.Translate("ITEM");
    else
        str = str + theTranslator.Translate("ITEMS");
    theGamesMenu.children[0].children[0].New__text_games01.text = str;
}
```

When the list was empty:

```javascript
if (titleSelect == -1)
{
    str = theTranslator.Translate("Go buy some games!");
}
```

The humor was there from the start.

### Per-Game Detail Screens

Select a game and you got a full detail screen (`Games_Title/default.xap`, 3,303 lines) with:

**Fixed menu items:**
- "New Game" (launches the title)
- "Saved Games" (`log("Saved games are not implemented yet!")` -- still in development)

**Up to 10 custom menu items per game**, defined in `menu.xbx`:
```javascript
for (i = 1; i <= 10; i = i + 1)
{
    t = menu.GetValue("Title" + i);
    if (t != "")
    {
        b = menu.GetValue("Body" + i);
        tbl = menu.GetValue("Table" + i);
        nws = menu.GetValue("News" + i);
        img = menu.GetValue("Image" + i);
        // must have one!
        if (tbl + nws + img == "")
        {
            log("Invalid custom title menu, must have news, table, or image!");
        }
    }
}
```

Games could ship custom dashboard entries -- "View High Scores," "Read News," "See Screenshots" -- each backed by images, text, or data tables stored as `.xbx` files in TDATA.

**Per-game news feeds**, localized per language:
```javascript
function GetTitleNews(titleID)
{
    info = new Settings;
    info.file = GetTitleDataPath(titleID) + "\\news.xbx";
    info.section = theTranslator.GetLanguageCode();
    return info;
}
```

**Publisher names and title metadata** from `meta.xbx`:
```javascript
info = GetTitleMeta(titleID);
str = "*" + info.GetValue("TitleName") + "*\r" + info.GetValue("PubName");
```

**Game cover art** loaded from UDATA:
```javascript
function GetTitleImage(titleID)
{
    return GetUserDataPath(titleID) + "\\title.xbx";
}
```

The file naming wasn't finalized -- `meta.xbx` became `TitleMeta.xbx`, `title.xbx` became `TitleImage.xbx`, `news.xbx` became `Info.xbx` by the next build.

### Animated Mechanical Arms

The detail screen had two sets of mechanical arms (`GM_L2_Arm01_*` and `GM_L2_Arm02_*`) that animated between the menu view and the info panel view. Each arm had multiple joints that moved independently, and the code checked `c.GM_L2_Arm01_04.moving` to prevent input during animation. The tubes were still in development: `children[0].GM_L2_tube_01.visible = false; // NYI`

### Hello Kitty Made It Into the Code

Seton Kim's REZN8 pitch deck featured "Hello Kitty trapped in an alien pod." The engineers took that literally. The game browser's title icon display uses:

```vrml
material MaxMaterial { name "HelloKittyIcon-trans" }
geometry DEF MV_L1podicon_starwars-FACES Mesh { url "MV_L1podicon_starwars-FACES.xm" }
```

Star Wars mesh name, Hello Kitty material name -- but it's more than just a name. The `HelloKittyIcon-trans` material with transparency is Hello Kitty's silhouette forming the outline of the game icon pod. Seton Kim pitched "Hello Kitty trapped in an alien pod" and the 3D artists modeled the pod border as Hello Kitty's outline framing the game art. The design concept made it all the way from the pitch deck into actual mesh geometry.

### Test Data and Developer Traces

The three test games on the dashboard team's devkit were revealed by the `GameName_*.tga` textures baked into the Memory_Files folder:

- **GameName_01.tga**: `HELLO KITTY CUBE FRENZY` -- a real PS1 game (1998). This is why the `HelloKittyIcon-trans` material exists. They were literally using Hello Kitty: Cube Frenzy as a test title, and the 3D artists took the IP and ran with it.
- **GameName_02.tga**: `STARCRAFT` -- Blizzard was actively porting it to Xbox in 2001. The dashboard team had a devkit build. It was eventually cancelled.
- **GameName_03.tga**: `DIABLO II` -- confirmed by 11 save game icon mockups (`SavedGameIcon_03-01` through `03-12`, skipping 07), all circular-cropped Diablo II gameplay screenshots with character levels overlaid (18, 26, 26, 29, 37, 37, 45, 57, 57, 63, 82). Blizzard had D2 running on Xbox devkits. Also never shipped.

Two cancelled Blizzard Xbox ports and a PS1 Hello Kitty game were the test data for Microsoft's dashboard.

The `Game_Icon_03.tga` (the title icon for Diablo II in the game browser) is a demonic red skull with Hello Kitty ears -- the `HelloKittyIcon-trans` transparency mask applied to Diablo art. The "Hello Kitty trapped in an alien pod" design concept taken to its extreme.

The `starcraft_image.tga` is not StarCraft art -- it's a face (possibly alien) framed by the Hello Kitty pod outline, the same transparency mask applied to whatever game art was being displayed.

Other textures in Games_Title reveal the planned per-game menu:
- **gamesoption.tga**: Menu mockup showing HIGH SCORES, NEW GAME, SAVED GAME, TIPS, UPDATES -- five options per game. "Tips" and "Updates" were planned features that never shipped.
- **highscores.tga**: A leaderboard with developer nicknames: ERGOMAN (1,200,000), SE7EN, JACKY, JON, JFFF, SPARKLES, DVDA, SUN ORBIT, REV SHARPTON, TONY, MANNIX. Eleven entries baked into a test texture.
- **hranking.tga**: "Check where you rank IN THE WORLD. Check where you rank AMONG YOUR FRIENDS." -- Online leaderboards were planned from April 2001, 19 months before Xbox Live launched in November 2002.

The background wallpaper (`background.bmp`) reads: "THIS VERSION IS FOR INTERNAL TESTING PURPOSES ONLY."

In the Memory folder: `MEM_L1_text_stevesgames-FACES.xm` AND `MEM_L2_text_stevesgames-FACES.xm` -- "Steve's Games" baked into mesh text on two separate screens. Someone named Steve was the test user on the dashboard team.

The Memory_Files meshes also contain `MEM_L2_hk_*` prefixed meshes -- "hk" for Hello Kitty. `MEM_L2_hk_panel_header`, `MEM_L2_hk_savedgame_pod_00`, `MEM_L2_hk_savedgame_row`. The Hello Kitty theming extended beyond the game browser into the save game viewer.

And line 744 of `games.xap`:

```javascript
// Reset the selection to the first item every time we enter
// this screen; this is probably bad for the user, but Victor
// made me do it... :-)
```

A developer overruled by someone named Victor on a UX decision, with a smiley to show no hard feelings. But it wasn't just this one comment -- Victor's name appears three separate times across the codebase:

```javascript
// games.xap:743 -- game list selection reset
// this is probably bad for the user, but Victor made me do it... :-)

// settings.xap:459 -- clock menu selection reset
nCurClockMenuItem = 0; // I hate this, but Victor made me do it...

// settings.xap:1335 -- root settings menu selection reset
nCurSettingsMenuItem = 0; // I hate this, but Victor made me do it...
```

The developer's tone escalated from "probably bad for the user :-)" to straight up "I hate this." Victor wanted every menu to reset to the top item when re-entered. The developer disagreed. Victor won. The comments survived through build 3729 (July 2001) into settings2.xap.

### The "Typesdsafsda" Origin

Line 543 of `games.xap`, April 2001:

```javascript
c.txtItem1.appearance.name = "Typesdsafsda";
c.txtItem2.appearance.name = "Typesdsafsda";
// ...8 items total
```

This is where it was born. A Microsoft developer needed a material name for deselected menu text and mashed their keyboard. Twenty-five years later, this exact string is still in UIX Lite's `harddrive.xap`, BSX's scripts, the Commemorative build, and every community dashboard that descended from these original scripts.

The same developer did it again one month later. Build 3521's music player (May 2001) introduced `Material #133sdsfdsf` -- they started typing the 3ds Max material number and smashed the keyboard for the suffix. Two keyboard-mash material names from the same person, one month apart.

The dashboard is full of raw 3ds Max material IDs that nobody renamed: `Material #133`, `Material #1334`, `Material #1335`, `Material #1336`, `Material #133511`, `Material #108`, `Material #10822`. The artists exported their scenes from 3ds Max and the auto-generated IDs went straight into production.

### The Five Main Menus

Build 3424 ships with two main menu versions side by side, and the retail dashboard uses a fifth iteration. Here's the full evolution:

**MainMenu (v1)** -- 926 lines. Simple, generic geometry. Uses `USE pod` / `USE podsocket_inner` / `USE podshell_00` instancing. Defines **five** menu items:

```
DEF SettingsItem Group   (line 263)
DEF MusicItem Group      (line 399)
DEF MoviesItem Group     (line 528)  // COMMENTED OUT
DEF MemoryItem Group     (line 634)
DEF GamesItem Group      (line 763)
```

The original plan was **five pods**: Settings, Music, Movies, Memory, Games. The Movies pod has a full 3D structure (pod, socket, support arms, shell, backing) but it's already wrapped in a comment block by the time build 3424 ships. Movies was the first casualty -- cut before the alpha even went out.

**MainMenu2 (v2)** -- 2,743 lines. Complete redesign. Camera noise effect (`MainCam_With_Noise`), `pod_rotate_structure` for selection animation, numbered podsupports, and `fade 0.25` on deselected items. **Four items** after Movies was dropped:

```
DEF theMemoryItem Transform { fade 0.25 ... }   (line 293)
DEF theMusicItem Transform { fade 0.25 ... }    (line 837)
DEF theGamesItem Transform { fade 0.25 ... }    (line 1372)
DEF theSettingsItem Transform { fade 0.25 ... } (line 1905)
```

The `game_*` prefix meshes first appear here: `game_arm01`, `game_podshell_*`, `game_nozzle*`, `gamepod_backing*`, `games_panel_support`, `games_metapanel`. 114 game-related references.

**MainMenu5 (v5)** -- 2,114 lines in 3729 (shorter than v2!). Inherits v2's structure but **drops theGamesItem**. Three items:

```
DEF theMusicItem Transform { fade 0.25 ... }    (line 129)
DEF theMemoryItem Transform { fade 0.25 ... }   (line 643)
DEF theSettingsItem Transform { fade 0.25 ... }  (line 1157)
```

All 92 `game_*` mesh references remain as dead 3D assets. The pod shells, nozzles, dials, arms, sockets, backing panels, tubes, and panel supports for the Games pod ship in every retail dashboard XIP from 3729 through 5960. People in the early scene -- the #xboxdash IRC crowd, the xboxdash.net regulars -- may well have known what these meshes were. But that knowledge got lost over 20 years as the scene went quiet and the people moved on.

| Version | Lines | Items | Built |
|---------|-------|-------|-------|
| MainMenu (v1) | 926 | Settings, Music, ~~Movies~~, Memory, Games (5, Movies commented out) | 3424 |
| MainMenu2 (v2) | 2,743 | Memory, Music, Games, Settings (4, Movies cut) | 3424 |
| MainMenu5 (v5) | 2,114 | Music, Memory, Settings (3, Games cut) | 3729-5558 |

Versions 3 and 4 existed between MainMenu2 and MainMenu5 but haven't been found in any recovered build.

**Movies text mesh survives.** `t_movies.xm` exists alongside `t_games.xm`, `t_memory.xm`, `t_music.xm`, `t_settings.xm` in the MainMenu/ folder -- 3D text labels for all five pods. But the second mesh export pass (`Main_*_text_*-FACES.xm`) only covers four: games, memory, music, settings. No `Main_movies_text_movies-FACES.xm`. Movies was being cut during the art pipeline itself.

Games had extra art: `Main_games_backing-FACES.xm` -- a dedicated backing panel with no equivalent for other items.

---

## Build 3521 -- May 2001 (Alpha II)

![Build 3521 -- 3-pod menu, Games removed](images/alpha-running/3521-mainmenu-3pods.png)
*Build 3521 running in UIX Desktop. One month after 3424, Games is gone from the main menu. Memory, Music, Settings.*

One month after 3424. The game launcher still exists, but Games has already been demoted.

```javascript
nCurMainMenuItem = 1; // start with music selected
```

And the main menu is already down to 3 items:
```javascript
if (nCurMainMenuItem < 2)
    nCurMainMenuItem = nCurMainMenuItem + 1;
```

Memory (0), Music (1), Settings (2). The `theGamesItem.SetRotation()` calls are commented out. `GoToGames()` still exists, `games.xap` still exists (794 lines, +28 from 3424), but nothing in the main menu calls it. Games went from default to dead in one month.

**XAP growth in one month (3424 -> 3521):**
| Script | 3424 | 3521 | Change |
|--------|------|------|--------|
| default.xap | 391 | 738 | +89% |
| games.xap | 766 | 794 | +4% |
| memory.xap | 590 | 820 | +39% |
| music.xap | 987 | 1,850 | +87% |
| settings.xap | 1,388 | 1,480 | +7% |

**New subsystems added:**
- `Keyboard/` (1,960 lines) -- virtual keyboard for text entry
- `Message/` (462 lines) -- dialog/message system
- `Music_PlayEdit/` (3,363 lines) -- music playlist editor (replaces simple Music_Play)
- `Music_Copy/` (1,582 lines) -- CD ripping interface

Still all loose files, still `.tga`/`.bmp` textures (no `.xbx` files on disc yet), still has `MainMenu/`, `MainMenu2/`, and an empty `NewMainMenu/` directory.

**.xbx file naming finalized.** Between 3424 and 3521, the game data file names were standardized:
- `news.xbx` -> `Info.xbx` (per-game news feed -- cut with game launcher)
- `menu.xbx` -> `Menu.xbx` (custom menu definition -- cut with game launcher)
- `meta.xbx` -> `TitleMeta.xbx` (title name/publisher -- survived to retail)
- `title.xbx` -> `TitleImage.xbx` (cover art -- survived to retail)

`Info.xbx` and `Menu.xbx` disappeared when the game launcher was killed. Only `TitleMeta.xbx` and `TitleImage.xbx` made it to retail -- the minimum the memory manager needed to show game names and icons. The `SaveMeta.xbx` and `SaveImage.xbx` per-save equivalents were added later.

**Sound effects added to game launcher.** The 3424 game browser was completely silent. By 3521, every action has sound: `PlaySoundA()`, `PlaySoundB()`, `PlaySoundMenuChange()`, `PlaySoundError()`, `PlayGoToSound()`, `PlayGoBackSound()`. The game launcher went from prototype to polished in one month -- and then was killed the same month.

---

## Build 3729 -- July 2001 (DVT3)

DVT3 = Design Validation Test, phase 3 -- late-stage hardware/software integration before manufacturing. The transitional build. Everything is in flux -- old and new systems running side by side. This is the moment the dashboard was being packed up for retail.

### The XIP Build Script

A file called `mkxips.cmd` sat in the XDASH directory -- Microsoft's actual XIP build script:

```batch
@echo off
..\xip\obj\i386\xip -q -m default.xip *.xap *.xt *.xm

cd MainMenu5
..\..\xip\obj\i386\xip -q -m -i GameHilite_01.bmp ..\MainMenu5.xip default.xap
cd..

cd Keyboard
..\..\xip\obj\i386\xip -q -m -i GameHilite_01.bmp ..\Keyboard.xip default.xap
cd..
```

The `-q -m` flags, the `-i GameHilite_01.bmp` include, the `xip.exe` tool living in `..\xip\obj\i386\` -- this is how every XIP was built. The community's `xiptool` reimplements what this script called.

### Loose Files AND XIPs Coexist

Both systems were running at the same time:

**Loose XAPs (10,884 lines total):**
| File | Lines |
|------|-------|
| `default.xap` | 1,021 |
| `games.xap` | 794 |
| `music.xap` | 2,135 |
| `memory.xap` | 1,116 |
| `memory2.xap` | 1,222 |
| `settings.xap` | 1,450 |
| `settings2.xap` | 2,249 |
| `dvd.xap` | 897 |

**XIPs packed alongside them:**
`default.xip`, `MainMenu2.xip`, `mainmenu5.xip`, `Keyboard.xip`, `Memory.xip`, `Memory2.xip`, `Memory_Files.xip`, `Music.xip`, `Music_PlayEdit.xip`, `Music_Copy.xip`, `Settings2.xip`, `Settings_Clock.xip`, `Message.xip`, `dvd.xip`

Both `MainMenu2.xip` and `mainmenu5.xip` exist -- the old four-pod menu and the new three-pod menu, shipping together. Both `Memory.xip` and `Memory2.xip` -- the old memory manager and its replacement. Both `settings.xap` and `settings2.xap` -- active iteration.

### Games Demoted from Default

```javascript
nCurMainMenuItem = 1; // start with music selected
```

Games dropped from the default selection. And the main menu now has only 3 items:

```javascript
if (nCurMainMenuItem < 2)
    nCurMainMenuItem = nCurMainMenuItem + 1;
```

Max index is 2, down from 3. Memory (0), Music (1), Settings (2). Games is gone from the main menu, but `games.xap` still exists as a loose file with the full game launcher code intact -- the `GoToGames()` function is defined, the Inline is wired up, the detail screens still work. It's just that nothing in the main menu calls it anymore.

### The Vertex Shader Source

`effect3.vsh` -- the actual NV2A vertex shader source code for the falloff glow effect that makes the dashboard look like it's underwater:

```
vs.1.0

// v0  -- position
// v3  -- normal
//
// c0-3   -- transpose world/view/proj matrix
// c5-8   -- inverse world/view matrix
// c10-13 -- transpose world/view matrix
// c15    -- front color
// c16    -- sideColor - frontColor

// transform position to screen in oPos
// transform normal to world space in r0
// transform position to world space in r3
dp4 oPos.x, v0, c0
dp4 r2.x, v0, c10
dp3 r0.x, v3, c5
...
// compute fall-off effect colors; D = normal dot position
dp3 r0.w, r0, r2
mov r1, c16
max r0.w, r0.w, -r0.w
mad oD0, r1, r0.w, c15
```

Normal dot position, clamped absolute value, lerp between front color and side color. That's the entire glow. The community documented this from IDA Pro disassembly of NV2A microcode -- turns out the source was sitting in a recovery disc the whole time.

### Five Ambient Soundscapes

Build 3424 had no ambient audio. Build 3729 introduced the atmospheric sound system:

```javascript
DEF theAmbientSounds Group
{
    children
    [
        DEF theAmbientSound1 AudioClip { url "AMB_01_THROB_LR.wav" loop true volume 0 fade 2 }
        DEF theAmbientSound2 AudioClip { url "AMB_12_HYDROTHUNDER_LR.wav" loop true volume 0 fade 2 }
        DEF theAmbientSound3 AudioClip { url "AMB_06_COMMUNICATION_LR.wav" loop true volume 0 fade 2 }
        DEF theAmbientSound4 AudioClip { url "AMB_10_ODYSSEY_LR.wav" loop true volume 0 fade 2 }
        DEF theAmbientSound5 AudioClip { url "AMB_05_ENGINEROOM_LR.wav" loop true volume 0 fade 2 }
    ]
}
```

Retail kept three: HYDROTHUNDER, COMMUNICATION, ENGINEROOM. The two that were cut:
- **AMB_01_THROB** -- a deep rhythmic pulse
- **AMB_10_ODYSSEY** -- spacey, 2001-inspired ambience

### Randomized Sound Events

The sound system used `PeriodicAudioGroup` nodes to trigger random ambient events:

```javascript
PeriodicAudioGroup
{
    period 60       // minimum seconds between sounds
    periodNoise 20  // random seconds added to period
    children
    [
        AudioClip { url "AmbientAudio/AMB_EC_Steam1.wav" }
        AudioClip { url "AmbientAudio/AMB_EC_Steam2.wav" }
        // ...7 steam hiss variants
    ]
}
```

Four groups: steam hisses (every 60-80s), alien voices (every 120-150s), comm chatter with stereo panning (every 10-20s), and a rare pinger/control room loop (every 300-360s). The comm voices pan left and right (`pan -100` / `pan 100`) to create the illusion of radio transmissions happening around you.

The comm voice clips are modified audio from the Apollo missions -- actual NASA recordings processed to sound like garbled transmissions from deep space. Combined with the steam hisses and alien voices, the dashboard sounds like a submarine or space station. It's atmospheric design that most people never consciously noticed but always felt.

### Music Credits on the Recovery Disc

Test music was baked into the recovery disc at `TDATA/fffe0000/music/`:

```
Breath of Giants courtesy of Jim Geist
Journey and Wilderness courtesy of Scott Snyder
Beauty Queen, Ham, Lead Head, Take Care of the Lonely courtesy of TrueHuman
```

Every devkit that ran this recovery disc got these seven WMA tracks in its music library.

### Language Files as Plain Text

Individual `.txt` files for each language (english.txt, french.txt, german.txt, italian.txt, japanese.txt, spanish.txt) with a simple key-value format:

```
# Main Menu Titles
MEMORY="MEMORY"
MUSIC="MUSIC"
SETTINGS="SETTINGS"

# Keyboard
DONE="DONE"
SYMBOLS="SYMBOLS"
SHIFT="SHIFT"
CAPS_LOCK="CAPS LOCK"
BACKSPACE="BACKSPACE"
```

With escape codes documented in the header: `\xA9` for copyright, `\u2122` for trademark, `\xA0` for non-breaking space. By retail, these were compiled into binary format inside the XIPs.

### Spinning Globe Timezone Selector

Build 3729's settings2.xap has a 3D globe (`GlobeIcon`) that was supposed to spin to show the selected timezone:

```javascript
/* TODO: Fix the calculation so the globe will spin the right time zone to the front...
        var n = 1.57 - nCurSettingsMenuItem * 6.283 / theTranslator.GetTimeZoneCount();
        c.GlobeIcon.children[0].children[0].SetRotation(0, 1, 0, n);
        c.GlobeIcon.children[0].rpm = 0;
*/
```

The math was wrong, so they gave up and just set `rpm = 4` to keep it spinning continuously. The globe spins in retail but never faces the right timezone.

### Parental Controls for Movies and Games

The settings2.xap (3729) still has parental control categories for both Movies and Games:

```javascript
theConfig.SetMoviePCFlags(nMoviePCLevel);  // MPAA ratings -- cut with Movies menu
theConfig.SetGamePCFlags(nGamePCLevel);    // ESRB ratings -- survived to retail
```

`SetMoviePCFlags()` was the MPAA rating equivalent that got cut along with the Movies menu. Separate parental control APIs for two content types, designed for a dashboard with five top-level sections. Retail only shipped with game ratings.

### Widescreen Being Tested

```javascript
DEF theScreen Screen
{
    width 640
    height 480
//  width 854
//  height 480
}
```

854x480 widescreen was commented out -- someone was testing it. The retail dashboard shipped 640x480 only.

---

## Build 3823 -- August 2001 (DVT3)

The cleanup build. The transition to XIPs is complete.

**What's gone:**
- All loose `.xap` files -- everything packed into XIPs
- `MainMenu2.xip` -- old four-pod menu dropped entirely
- `mkxips.cmd` -- build script removed from distribution
- `effect3.vsh` -- vertex shader compiled into the XBE
- `cellwall.bmp`, `GameHilite_01.bmp` -- textures moved into XIPs
- Language `.txt` files -- compiled into binary format
- All 5 ambient sound WAV files as loose files
- Main menu transition audio ("Global Main MenuBack/Fwd" WAVs)

**What's new:**
- `xboxlogo.bmp` / `xboxlogow.bmp` -- fallback logo for games without icons (wide and standard)
- `dvdstopw.bmp` -- widescreen DVD stop screen
- Both `Memory_Files.xip` AND `memory_files2.xip` -- still transitioning

**XIPs are dramatically bigger:**
The settings XIPs roughly doubled in size (e.g., `settings_list.xip` went from ~2.9MB to 9.5MB). The meshes and textures that were loose files in 3729 are now packed inside.

The ISO was titled "august2001betarecovery" with the warning "(don't run on final kits)" -- this was for DVT3 devkits only.

**Inside the XIP: games.xap is dead code.**
The `default.xip` in 3823 still contains `games.xap` (19,581 bytes) -- byte-for-byte identical to the version in 3729's XIP AND the loose file. The code was packed into the archive untouched, even though nothing calls it. Dead weight carried forward.

**ODYSSEY soundscape cut.**
The ambient sounds went from 5 to 4: THROB, HYDROTHUNDER, COMMUNICATION, ENGINEROOM. AMB_10_ODYSSEY_LR.wav was the first casualty.

**Per-area ambient tracks added.**
Each menu section gets its own soundscape via `AmbientAudioOn("MAIN MENU")`. The comments map them: 0=main menu, 1=memory, 2=music, 3=settings.

**Comm chatter dialed way back.**
The period increased from every 10-20 seconds to every 60-95 seconds. The Apollo mission comm voices were too frequent in 3729 -- you'd hear garbled NASA transmissions every few seconds. By 3823 they're a rare event.

**Volume normalization.**
All sounds got explicit volume levels for the first time: steam hisses at 0.83, alien voices at 0.83, comm chatter at 0.80, pinger/control room at 0.88. Previously everything was at the default (1.0).

**XAP version numbering incremented.**
Scripts transitioned from `memory.xap` + `memory2.xap` to `memory3.xap`, from `music.xap` to `music2.xap`, from `settings.xap` + `settings2.xap` to `settings3.xap`. The version numbers embedded in the filenames show the iteration history. A `settings_list.xap` (2,428 lines) briefly appeared -- a dedicated list UI component that was folded back into settings3.xap by the next build.

---

## Build 3911 -- August 2001 (DVT3)

The near-retail build. One month before the November launch.

**What changed from 3823:**
- `Memory_Files.xip` gone (old version), only `memory_files2.xip` remains
- XIP sizes settled to near-retail values (default.xip back to ~1.5MB)
- Menu transition sounds stripped down to just Memory Game Sub Menu In/Out

The XIP set from 3911 is essentially the retail set. Files like `mainmenu5.xip` (2,283,130 bytes) are byte-for-byte identical to what shipped in build 4604 a year later.

---

## Build 3944 -- November 2001

**Day one retail launch.** The dashboard that shipped in every Xbox sold at launch. Games menu: gone. Seven months between "games is default" in 3424 and "games doesn't exist" in the retail box.

---

## Build 4604 -- May 2002

The first post-launch dashboard with Xbox Live support. This is also a full XDK package with a complete filesystem dump, so we can see everything.

### Copyright Header Appears

Between 3911 and 4604, Microsoft added `// Copyright (c) Microsoft Corporation. All rights reserved.` to the top of `default.xap`. The pre-retail builds had no copyright notice in the scripts.

### THROB Soundscape Cut

The ambient sounds lost another track. `AMB_01_THROB_LR.wav` was replaced by a second instance of `AMB_12_HYDROTHUNDER_LR.wav` -- leaving only three unique soundscapes: HYDROTHUNDER (main menu + memory), COMMUNICATION (music), ENGINEROOM (settings). A new `pause_on_moving true` flag was added to pause ambient audio during menu transitions.

The ambient sound evolution:
| Build | Soundscapes |
|-------|------------|
| 3424 | None |
| 3729 | THROB, HYDROTHUNDER, COMMUNICATION, ODYSSEY, ENGINEROOM (5) |
| 3823 | THROB, HYDROTHUNDER, COMMUNICATION, ENGINEROOM (4 -- ODYSSEY cut) |
| 4604+ | HYDROTHUNDER x2, COMMUNICATION, ENGINEROOM (3 unique -- THROB cut) |

### Dual Dashboard Systems

The C drive contains BOTH the 3D XAP dashboard AND the flat `.mnu` devkit menus:

```
C:/
    xboxdash.xbe         -- 3D dashboard (what consumers see)
    xshell.xbe           -- devkit launcher

    default.xip          -- XAP dashboard
    mainmenu5.xip        -- main menu
    Memory2.xip          -- memory manager
    music2.xip           -- music player
    settings3.xip        -- settings
    ...19 total XIPs

    menus/               -- .mnu flat menus (devkit only)
        root.mnu
        settings.mnu
        machine.mnu
        online.mnu       -- Xbox Online welcome (uses seanhead.bmp!)
        eeprom.mnu
        ...33 total .mnu files

    images/
        seanhead.bmp     -- a developer's headshot
        background1.bmp  -- XDK background
        button_a.bmp     -- controller button overlays
        ...
```

The `.mnu` files reference code constants: `// Must be identical to KEYBOARD_MACHINENAMEHEADER_YPOS in 'constants.h'` -- tying the menu layout to the C++ source. And `online.mnu` displays `seanhead.bmp` -- a developer named Sean whose face was the Xbox Online welcome screen in the XDK launcher.

### XODASH Appears

A new directory: `XODASH/` with `xonlinedash.xbe` and `update.xbe`. This is the Xbox Live dashboard -- the separate binary that handled online sign-in, friend lists, and system updates. It shipped alongside the main dashboard as a separate executable.

### XIP Files Frozen

The XIP set is byte-for-byte identical between 4604 and 4627 (June 2002). Microsoft wasn't touching the dashboard scripts -- only the XBE binaries and support files changed between these builds.

### xlaunch.ini

The XDK launcher configuration, stored as UTF-16LE:

```ini
Language="English"
SelectedProg=""
LastGamertagCreated=""
SelectedMemoryUnitId=0x00000000
SafeArea="false"
HDTVSafeArea="false"
ExpandedView="false"
NoIconView="false"
```

### Ambient Audio Set Finalized

The full retail ambient audio set: 3 soundscapes (ENGINEROOM, COMMUNICATION, HYDROTHUNDER), 7 steam hisses, 13 alien voices, 9 comm voice clips with 4 static bursts, and a pinger with two control room loop variants. 39 files total. This exact set shipped through build 5960.

---

## Build 4627 -- June 2002 (DVT4)

Dashboard XIPs: **byte-for-byte identical to 4604.** The XBE binaries updated, the scripts didn't change. Includes `complex_4627debug.bin` -- a 1MB debug BIOS binary.

---

## Build 5558 -- June 2003

The first build with measurable dashboard changes since launch.

**XIP changes from 4604:**
| XIP | 4604 | 5558 | Delta |
|-----|------|------|-------|
| default.xip | 1,519,348 | 1,544,575 | +25KB |
| mainmenu5.xip | 2,283,130 | 2,300,516 | +17KB |
| memory_files2.xip | 2,619,139 | 2,708,134 | +89KB |
| settings3.xip | 2,808,774 | 2,808,705 | -69B |
| settings_language.xip | 1,425,470 | 1,429,750 | +4KB |
| Message.xip | 801,448 | 801,609 | +161B |

The dashboard was being actively updated again -- probably for Xbox Live features. The install instructions for the extracted files said:

> Simply copy the files over. Why doesn't someone (Slayer) write an evox.ini file to do this??

A developer calling out someone named Slayer to automate the deployment with an EvoX configuration file.

---

## Build 5933 -- February 2004

**Not a dashboard update.** Contains only `xshell.xbe` (devkit launcher), `dashboard.xbx` (a pointer file telling the XDK launcher where to find the test dashboard: `\Device\Harddisk0\Partition2\xshell.xbe`), and Xbox Live certification tools.

The cert tools include `CertFriend.xbc` and `CertInvite.xbc` for testing friend requests and game invites on PartnerNet (Microsoft's test Xbox Live environment).

---

## Hulkdash

A test ISO built by the community to boot a dashboard in xemu (XQEMU). Instructions:

> Delete default.xbe. Copy xboxdash/xboxdash.xbe to root and rename to default.xbe. Copy the below listed 4920 dashboard files into xboxdash/. Compile as ISO with Extract-XISO v2 and boot with XQEMU.

Based on 4920 dashboard files with one unique addition: `settings_adoc.xip` (4.2MB) -- not found in any other build. Likely related to the Xbox Live documentation/agreement system.

---

## The Full Timeline

| Build | Date | Phase | Key Changes |
|-------|------|-------|-------------|
| 3146 | Dec 2000 | Alpha I | `.mnu` flat menus, BMP buttons. Devkit setup utility. |
| 3223 | Feb 2001 | Alpha II | Added MU formatting. Still flat menus. |
| 3424 | Apr 2001 | Alpha II | **Full 3D XAP dashboard.** Loose files. Games is default. Complete game launcher with browser, detail screens, news, high scores, custom menus, mechanical arms, Hello Kitty pod outlines. Test data: Hello Kitty Cube Frenzy, StarCraft, Diablo II. Developer high scores with nicknames. Online leaderboard mockups. `MainMenu/` (5 pods, Movies commented out) and `MainMenu2/` (4 pods) side by side. |
| 3521 | May 2001 | Alpha II | Games demoted from default. Menu down to 3 items. Keyboard, Message, Music_PlayEdit, Music_Copy added. default.xap nearly doubled. |
| 3729 | Jul 2001 | DVT3 | **Transition build.** Loose XAPs + XIPs coexist. `mkxips.cmd` build script. `effect3.vsh` vertex shader source. Both `MainMenu2.xip` and `mainmenu5.xip`. 5 ambient soundscapes (Apollo mission comm clips, 2 cut from retail). Music test tracks. Widescreen tested. File names finalized (TitleMeta.xbx, TitleImage.xbx, Info.xbx). |
| 3823 | Aug 2001 | DVT3 | **Full XIP transition.** All loose XAPs gone. MainMenu2 dropped. Language files compiled. Vertex shader embedded. `xboxlogo.bmp` fallback icon. |
| 3911 | Aug 2001 | DVT3 | **Near-retail.** Old Memory_Files.xip removed. XIP sizes settled. Many XIPs byte-for-byte identical to retail. |
| 3944 | Nov 2001 | **Retail launch** | Day one. Games menu gone entirely. |
| 4604 | May 2002 | Post-launch | First Xbox Live support. XODASH appears. `.mnu` devkit menus still ship alongside XIPs. `seanhead.bmp` in XDK images. |
| 4627 | Jun 2002 | DVT4 | XIPs identical to 4604. Debug BIOS included. |
| 4920 | -- | Retail | Standard community modding base. |
| 5558 | Jun 2003 | -- | First dashboard XIP changes since launch (+25KB default, +89KB memory). |
| 5933 | Feb 2004 | -- | XDK update only (devkit launcher + Live cert tools). No dashboard changes. |
| 5960 | -- | Final retail | XIP signature checks tightened. Last official dashboard. |

### The Death of games.xap

| Build | Date | Status |
|-------|------|--------|
| 3424 | Apr 01 | Full game launcher. Games is default menu item (item 2 of 4). 766 lines + 1,547 line browser + 3,303 line detail screen. |
| 3521 | May 01 | Games removed from main menu. Down to 3 items. Music is new default. `GoToGames()` still exists, `games.xap` grows to 794 lines, but nothing calls it. **One month from default to dead.** |
| 3729 | Jul 01 | Still exists as loose file (794 lines, unchanged). Same dead code in default.xip. |
| 3823 | Aug 01 | Loose file gone. 19,581 bytes packed into default.xip, byte-for-byte identical to 3729 copy. Dead weight in the archive. |
| 3911 | Aug 01 | Fully stripped from XIPs. Zero game launcher code remains. |
| 3944 | Nov 01 | Retail ships without it. |

Born April 2001. Demoted May 2001. Dead code through August 2001. Then 20+ years of community workarounds.

![Build 3424 memory manager with 4 controller pods](images/alpha-running/3424-memory-controllers.png)
*Build 3424 memory manager. The Xbox console in the center orb, four controller pods with memory unit slots, TOTAL MEMORY / AVAILABLE MEMORY panel. Darker aesthetic than retail -- the original submarine/space station vibe.*

### SavedGameGrid Could Always Launch Games

The community didn't discover a hidden feature -- SavedGameGrid could launch games from day one. Build 3424's `memory.xap` (April 2001):

```javascript
theTitleCollection.LaunchByID(c.theSavedGameGrid.GetTitleID(nTitle), "");
```

The memory manager and the game launcher were parallel systems. The game launcher used `TitleCollection` directly. The memory manager used `SavedGameGrid` for save files, but exposed `GetTitleID()`, `GetTitleName()`, `curTitle`, `selectUp()`, `selectDown()` because saves are organized by title. Pressing A on a title launched the game.

When the game launcher was cut, `SavedGameGrid` survived with its title enumeration intact. The community later abused it for icon loading -- repurposing the save game icon display to show game cover art. The actual game launching was done through entirely different hacks in other XAPs (music player hijack, XBE path construction, Settings INI workarounds).

### What the Community Rebuilt

Everything else in build 3424's game launcher was rebuilt piecemeal over two decades using exploits across different XAPs:

- **Game launching**: Music player hijack to execute arbitrary XBEs (see [XAP Scripting](xap-scripting.md))
- **Game icons**: SavedGameGrid's icon loader repurposed for cover art display
- **Game database**: INI file reader (`Settings` object + `.xbx` text files) as metadata storage
- **Custom menus**: Settings panel framework for game categories
- **XBE path construction**: NT kernel device paths managed in JavaScript

People with alpha kits and devkits may have seen the game launcher running -- but back then, that kind of thing stayed in screenshots and IRC snippets rather than shared files. The retail XIPs had the dead meshes, the REZN8 engineering renders showed the concept, and the community had enough pieces to know something had been cut. What got lost over the years was the full picture: the scripts, the test data, the per-game news feeds, the five-pod menu with Movies. These are rediscoveries, not first discoveries -- putting the story back together from recovery discs that have been floating around for decades.

### XAP Script Evolution (lines of code in default.xip)

| Build | default.xap | dvd.xap | memory | music | settings | games.xap |
|-------|------------|---------|--------|-------|----------|-----------|
| 3424 | 391 | -- | 590 | 987 | 1,388 | 766 (loose) |
| 3729 | 1,021 | 897 | 1,222 (v2) | 2,110 | 2,249 (v2) | 794 (loose + XIP) |
| 3823 | 1,331 | 987 | 1,389 (v3) | 2,550 (v2) | 5,238 (v3) | 19,581B dead code in XIP |
| 3911 | 1,370 | 1,020 | 1,491 (v3) | 2,588 (v2) | 5,360 (v3) | **Removed** |
| 4604 | 1,502 | 1,151 | 1,523 (v3) | 2,615 (v2) | 3,838 (v3) | -- |
| 5558 | 1,715 | 1,163 | 1,732 (v3) | 2,615 (v2) | 4,031 (v3) | -- |

The version numbers in the filenames tell the story: `memory.xap` became `memory2.xap` became `memory3.xap`. Settings went through at least three major rewrites (1,388 -> 2,249 -> 5,238 -> 3,838 lines). Music doubled in size and then froze -- no changes between 4604 and 5558.

The `default.xap` root scene more than quadrupled from 391 lines in the alpha to 1,715 in the final builds, absorbing Xbox Live hooks, popup management, memory unit enumeration blocking, and the ambient audio system that didn't exist in 3424.

### The .xbx Extension Trick

The `.xbx` extension was used for both binary textures (XPR packed resources like `titleimage.xbx`, `saveimage.xbx`) and plain text metadata files (like `TitleMeta.xbx`, `SaveMeta.xbx`). The text files were UTF-16LE key-value pairs:

```
TitleName=Xbox Dashboard
```

The binary texture files started with `XPR0` (0x58505230) -- Xbox Packed Resource format.

Microsoft signed the XIPs but never signed the loose `.xbx` files on the hard drive. The community discovered that text `.xbx` files -- which used the same extension as signed textures -- were writable, readable by the dashboard's `Settings` object, and could store arbitrary data. This was the foundation for every dashboard mod: if you could write a `.xbx` text file, you could feed data to the dashboard scripts.

The irony is that the `.xbx` text convention wasn't designed for the memory manager. It was designed for `games.xap` -- `Info.xbx` for news feeds, `Menu.xbx` for custom menus, `TitleMeta.xbx` for publisher data. The game launcher was cut, but the writable unsigned text file format it introduced lived on. Twenty years of community dashboard mods were built on the ghost of a feature Microsoft killed before the console shipped. The full exploit chain is documented in [XAP Scripting](xap-scripting.md).
