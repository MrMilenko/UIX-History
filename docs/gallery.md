# Screenshots and History

A visual timeline of UIX development -- from broken early Theseus renders to working skins, community XIP experiments, and features Microsoft never shipped.

## The Broken Early Days

When the Theseus decompilation was still figuring out transforms and mesh rendering, screenshots like this became memes in the Xbox-Scene Discord:

![Early broken Theseus render](../images/theseus_early_broken.jpg)
*The orb is inside out, the panels are flying off at wrong angles, but the text renders and the XAP VM is running. This is the "I can see this" stage -- proof that the scripting engine works, even if everything else is wrong.*

## Alpha Dashboards Running in UIX Desktop (2026)

Alpha dashboard builds from 2001 loaded in the UIX Desktop engine on macOS. These builds have been run on devkits and alpha hardware before, but nobody had really dug into the scripts and meshes to see what was hiding in them.

![Build 3424 -- 4 pod main menu with GAMES](images/alpha-running/3424-mainmenu2-4pods.png)
*Build 3424 (April 2001). The MainMenu2 with four pods: Memory, Music, Games, Settings. Games is selected by default. This menu was cut to three pods by the next month.*

![Build 3424 -- Game launcher](images/alpha-running/3424-game-launcher.png)
*The "GAMES" browser from build 3424. Eight-item scrollable list, dedicated 3D pod, mechanical arm, info panel. Cut in August 2001.*

![Build 3424 -- Memory manager](images/alpha-running/3424-memory-controllers.png)
*Build 3424 memory manager. Xbox console in the center, four controller pods with memory unit slots. Darker aesthetic than retail.*

![Build 3521 -- 3 pod menu](images/alpha-running/3521-mainmenu-3pods.png)
*Build 3521 (May 2001). One month later, Games is gone. Memory, Music, Settings.*

## Alpha Test Data

Textures found on the build 3424 recovery disc, April 2001:

![Test game names](images/alpha-3424/GameName_03.png)
*DIABLO II -- one of three test games on the dashboard team's devkit.*

![Hello Kitty demon skull](images/alpha-3424/Game_Icon_03.png)
*The game icon for Diablo II: a demonic skull with Hello Kitty ears. The "Hello Kitty trapped in an alien pod" design concept taken to its extreme.*

![Developer high scores](images/alpha-3424/highscores.png)
*Test leaderboard with developer nicknames: ERGOMAN, SE7EN, JACKY, JON, JFFF, SPARKLES, DVDA, SUN ORBIT, REV SHARPTON, TONY, MANNIX.*

![Online leaderboard mockup](images/alpha-3424/hranking.png)
*"Check where you rank IN THE WORLD. Check where you rank AMONG YOUR FRIENDS." Online leaderboards planned 18 months before Xbox Live launched.*

![Per-game menu mockup](images/alpha-3424/gamesoption.png)
*Planned per-game detail menu: HIGH SCORES, NEW GAME, SAVED GAME, TIPS, UPDATES. "Tips" and "Updates" never shipped.*

## Microsoft's Hidden Game Launcher

Found in alpha XIP archives -- a complete game launcher with scroll arrows, list panel, and dedicated 3D art:

![Internal game launcher from alpha XIPs](../images/classic_internal_launcher.png)
*The "GAMES" browser found in pre-release XIP archives. The early #xboxdash IRC crowd may have seen this, but the knowledge was lost by the time the community started rebuilding game launchers from scratch.*

## Theseus on xemu

Theseus running in [xemu](https://xemu.app) (the open-source Xbox emulator) with the UIX Lite skin:

![Theseus in xemu - green skin](../images/image8.png)
*Theseus running in xemu with the XAP editor inspecting material properties. The circled text shows the MaxMaterial node being examined live.*

## UIX Lite Skins

The skin system in action -- community skins completely reshaping the dashboard's appearance while running on the same engine:

![Blue skin with orb menu](../images/unknown.png)
*UIX Lite blue skin running in xemu -- Memory, Music, Hard Drive, Settings, Skins menu items.*

![Green skin with dotted menu items](../images/unknown3.png)
*A community skin with custom menu panel styling.*

![Halo character skin](../images/unknown6.png)
*Community skins could replace the entire visual identity -- meshes, textures, backgrounds, everything.*

## The Orbs Menu

A radial menu layout that arranged game categories around a central hub -- Applications, Dashboards, Games, Emulators, and more:

![Orbs menu - radial layout](../images/1unknown.png)
*The "ORBS" menu showing category tabs: Main, Games, Apps, Emus, Dash, Hdd, Info.*

![Orbs menu - category selection](../images/image01.png)
*Category view with Xbox Hard Drive and Games Menu panels.*

![Orbs menu - circular layout](../images/3unknown.png)
*An alternate radial layout arranging Games, Music, Saves, Config, Skins as floating orbs.*

## Skin Info System

Skin metadata read from unsigned `.xbx` text files using the `Settings` object:

![Skin info dialog](../images/2unknown.png)
*Author: TeamUIX, Date: 10-31-04, Version: 1.0, Made with: MS Presets/Photoshop, Credits: foood.net for DVD icons. All stored in an unsigned .xbx text file.*

## Music Player

An older version of the music player UI:

![Music player - soundtrack management](../images/4unknown.png)
*"Fighting Mix" -- an Xbox SDK sample audio set. Play, Copy, Edit, Remove options for soundtrack management.*

## Title ID Enumeration

Early testing of the title enumeration system showing raw title IDs being read from the hard drive:

![Title ID display](../images/image12.png)
*6 titles found, displaying raw title IDs (080299ff, fff051f, faadf32f, etc.) alongside the Xbox logo. The "APPLICATIONS" category label visible at bottom.*

![Game titles with icons](../images/unknown7.png)
*2 Titles: Conker: Live and Reloaded, Halo -- with the Xbox logo icon loaded from SavedGameGrid. The "GAMES" category working.*

## File Browser

The Theseus file browser running in xemu, showing the C: drive contents -- including backup XBE files from the decompilation process:

![File browser in xemu](../images/unknown2.png)
*Source: C: showing README.txt, Xboxdash.xbe, Xboxdash.xbe-backup, Xboxdash.xbe-broke. Destination panel showing font files. This is the file management system built on top of Microsoft's settings panel UI.*

## Xbox Live Account Selection

Theseus's Xbox Live account selection running against Insignia:

![Xbox Live account selection](../images/Screenshot_2023-11-28_at_1.07.43_AM.png)
*SELECT ACCOUNT showing "Avalaunch" and "dvd2xbox" -- placeholder account names used during early testing before the actual Xbox Live account format was figured out.*

## More Theseus Testing

![Early menu testing](../images/image0.png)
*Menu panels rendering but the orb background is still broken. Settings appearing twice -- a script loading bug being worked through.*

![Orbs menu test](../images/unknown12.png)
*Testing the Orbs menu layout -- Memory, Music, Hard Drive, Settings, Orbs Menu items.*
