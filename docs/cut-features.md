# Cut Features: What Microsoft Built and Scrapped

## The Game Launcher That Never Shipped

Found in alpha XIP archives: `games.xap` -- a 766-line game launcher with features that went far beyond what the community reimplemented years later. Microsoft didn't just consider a game launcher -- they built one, tested it, and pulled it before retail.

![Microsoft's cut game launcher running in Theseus](../images/classic_internal_launcher.png)
*The alpha "GAMES" browser loaded in Theseus -- scroll arrows, selection highlight, dedicated panel layout. This existed in Microsoft's XIP archives and never shipped.*

The key sections of the script are documented below. See **[XAP Scripting](xap-scripting.md)** for how the community rebuilt these features years later using exploits in the retail dashboard's scripting system.

### What It Did

**Title browsing** with an 8-item scrollable list, scroll arrows, and a selection highlight:

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

    UpdateTitleListText();
}

function UpdateTitleListText()
{
    var titleCount = theTitleCollection.GetTitleCount();
    for (var i = 0; i < 8; i = i + 1)
    {
        if (titleScroll + i >= titleCount)
            c.children[i].children[0].geometry.text = "";
        else
        {
            var str = theTitleCollection.GetTitleName(titleScroll + i);
            c.children[i].children[0].geometry.text = str.substr(0, 60);
        }
    }
}
```

One API call to get a title name. One line to display it. The `TitleCollection` node handled enumeration natively -- no folder scanning, no caching, no INI file databases.

**Per-game detail panels** with title info, publisher names, and news:

```javascript
// no news.xbx (or it's empty) -- display the title and publisher
info = GetTitleMeta(titleID);
str = "*" + info.GetValue("TitleName") + "*\r" + info.GetValue("PubName");
```

Microsoft's `TitleMeta.xbx` was designed to carry publisher names. The retail version only stored the title name. The publisher field was planned, implemented in the launcher UI, and then the entire launcher was cut.

**Per-game news feeds** read from localized `.xbx` files:

```javascript
function GetTitleNews(titleID)
{
    info = new Settings;
    info.file = GetTitleDataPath(titleID) + "\\Info.xbx";
    info.section = theTranslator.GetLanguageCode();
    return info;
}

function FormatNews(info, nMax)
{
    var str = "";
    for (var i = 1; i <= nMax; i = i + 1)
    {
        var t = info.GetValue("Title" + i);
        var b = info.GetValue("Body" + i);
        if (t != "") str = str + "*" + t.substr(0, 60) + "*\n";
        if (b != "") str = str + b.substr(0, 250) + "\n";
    }
    return str;
}
```

Each game could have an `Info.xbx` file in its TDATA directory with up to 10 news entries, localized per language. This was a game-specific news feed system -- in 2001, before Xbox Live even existed. Someone at Microsoft was thinking about per-game content updates delivered via the hard drive.

**Custom menu extensions per game** -- developers could add up to 10 custom menu items with images, text panels, or data tables:

```javascript
function SetupTitle()
{
    var menu = GetTitleMenu(curTitleID);
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
                break;
            }
        }
    }
}
```

Games could ship a `Menu.xbx` file with custom dashboard entries -- "View High Scores," "Read News," "View Screenshot" -- each backed by images, text, or data tables stored as `.xbx` files. This was a game-dashboard integration system years before Xbox 360's blade interface or PS4's info panels.

**The empty state message:**

```javascript
if (titleSelect == -1)
{
    str = theTranslator.Translate("Go buy some games!");
}
```

Even the humor made it into the alpha build.

**The test games on the dashboard team's devkit** (identified from `GameName_*.tga` textures on the recovery disc):
- Hello Kitty: Cube Frenzy (a PS1 game used as test data on devkits -- this is why the `HelloKittyIcon-trans` material exists in the dashboard meshes)
- StarCraft (Blizzard was porting it to Xbox in 2001, eventually cancelled)
- Diablo II (11 save game icon mockups are Diablo II gameplay screenshots with character levels)

**Per-game menu mockup** (`gamesoption.tga`) shows five planned options: HIGH SCORES, NEW GAME, SAVED GAME, TIPS, UPDATES. "Tips" and "Updates" never shipped.

**Online leaderboards** (`hranking.tga`): "Check where you rank IN THE WORLD. Check where you rank AMONG YOUR FRIENDS." Planned in April 2001, 19 months before Xbox Live launched in November 2002.

**Developer high scores** (`highscores.tga`): A leaderboard with test entries -- ERGOMAN (1,200,000), SE7EN, JACKY, JON, JFFF, SPARKLES, DVDA, SUN ORBIT, REV SHARPTON, TONY, MANNIX.

### The 3D UI

The launcher had its own 3D environment with dedicated meshes:

```vrml
tunnel Transform
{
    translation -139.800003 0.000000 -99.430000
    children [
        Shape {
            appearance Appearance { material MaxMaterial { name "Tubes" } }
            geometry Mesh { url "Games/Tunnel_01-FACES.xm" }
        }
    ]
}
```

Dedicated tunnel meshes (`Games/Tunnel_01-FACES.xm`, `Games_Title/Tunnel_02-FACES.xm`), multiple viewpoints for the title list and detail views, and animated mechanical arms (`GM_L2_Arm01_04`, `GM_L2_Arm02_05`) that moved between menu states. This wasn't a prototype mockup -- it had finished 3D art assets.

### What the Community Rebuilt

When the community built `harddrive.xap`, they had to:
- Hijack the music player's visual assets (no dedicated game launcher meshes existed in retail)
- Build title browsing from scratch using `Folder` nodes meant for soundtrack directories
- Repurpose `SavedGameGrid` as an icon loader (Microsoft's launcher used `GetTitleImage()` directly)
- Implement their own caching system with pipe-delimited INI files (Microsoft's version just called `theTitleCollection.GetTitleName()`)
- Skip per-game news, publisher info, and custom menus entirely (cut from retail, with only remnants in the XIP meshes and audio)

The alpha launcher loaded icons with one line:
```javascript
theGamesMenu.children[0].children[0].TitleTexture.url = strImageURL;
```

The community's version needed 15 lines of `SavedGameGrid` wiring to achieve the same thing, because the `TitleTexture` approach wasn't available in the retail dashboard's node set.

### The Meshes Survived

The game launcher's 3D assets are still sitting in `MainMenu5.xip` in every retail dashboard: `game_podshell_*`, `game_arm*`, `game_nozzle*`, `games_tube*`, `games_metapanel-FACES.xm`. The transition audio survived too: "Games Main Menu In/Out", "Games Sub Menu In/Out", "Games Info Screen In/Out" -- all present in the retail audio directories.

These meshes and sounds shipped in every retail dashboard through 5960. Microsoft never cleaned them out. The early scene may have known what they were -- people were digging into XIPs from the start -- but that context got lost as the years passed. By the time the community was rebuilding game launchers in the 2010s and 2020s, the connection to the original alpha scripts had been forgotten.

### Why It Was Cut

It wasn't a hard cut -- it was a merge. The title browsing functionality got absorbed into `SavedGameGrid` and `TitleCollection`. In the retail dashboard, `SavedGameGrid` exposes `GetTitleCount()`, `GetTitleName()`, `GetTitleID()` -- the same data the games launcher would have needed. The system handles both save game enumeration AND title browsing, indexed by device unit (unit 8 = hard drive).

The dedicated launcher UI (the `games.xap` with its own 3D viewpoints, tube meshes, and mechanical arms) was the part that got pulled. The underlying system for enumerating titles on disk lived on as part of the memory manager.

BigJx's SavedGameGrid icon loading trick takes on a different light knowing this. When he wired the save game browser's `setSelImage()` function into the hard drive launcher to load game icons, he was reaching back into the title enumeration system that was originally built for game browsing -- just accessing it through the memory manager's door instead of the launcher door that Microsoft closed.

The `theTitleCollection` references (commented out but still present in alpha scripts) suggest the data source changed at least once during development. The `// NYI` (Not Yet Implemented) comments on the tube meshes and `"Saved games are not implemented yet!"` log message suggest it was still in active development when it was pulled.

Seton Kim's REZN8 design concepts show that game browsing was part of the original dashboard vision. The UI made it through concept, design, 3D art, audio, and scripting before being cut in the summer of 2001.

The community rebuilt it from scratch, using the music player's visual assets because they didn't know the game launcher's own meshes were sitting right there in MainMenu5.xip the whole time.

## The Movies Pod

The game launcher wasn't the only thing that got cut. The original MainMenu v1 (build 3424) was designed with **five** top-level sections: Games, Music, Movies, Memory, Settings. The Movies pod had full 3D geometry -- pod sphere, socket inner/outer, four support arms, mount, three shell pieces, and a backing panel -- all positioned and ready to render. But it was commented out before the alpha even shipped.

No `movies.xap` has been found in any recovered build -- though that doesn't mean one was never written. The Movies pod was likely planned as a video content browser separate from the disc-triggered DVD player. It was the first casualty of the dashboard's scope reduction. By MainMenu2 (also in build 3424), Movies was gone entirely and the dashboard was down to four pods.

The Movies pod was uncommented and rendered in UIX Desktop in 2026 by adding a text label and re-enabling the highlight in the Switch node. The geometry Microsoft built in 2001 renders correctly -- they just never shipped it.
