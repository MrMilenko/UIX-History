# UIX History

> *Most people forgot what TeamUIX did. But some of us remembered.*

The original Xbox dashboard was a full 3D application built on Direct3D 8, with its entire UI defined in editable script files. Microsoft never intended anyone outside their team to touch it. They didn't document the formats, didn't publish the tools, and signed the archives with crypto.

Then someone noticed the text metadata files used the same extension as textures. And they weren't signed.

This is the story of what happened next.

---

## The Story

- **[Reverse Engineering the Xbox Dashboard](docs/reverse-engineering.md)** -- From hex editors in 2003 to IDA Pro and Ghidra during COVID. How the modding scene grew, went quiet, and came back. How UIX Lite brought community mods to the final retail dashboard. How Theseus grew from "can I load a XIP" into a standalone engine.

- **[The Lineage](docs/lineage.md)** -- tHc, BSX, UIX, UIX Lite, Theseus, UIX Desktop. How rival groups forked, fought, merged, and built on each other's code for 20 years.

---

## Xbox Internals

- **[Understanding the Xbox Dashboard](docs/understanding-dashboard.md)** -- How the retail dashboard works, from power-on to green orb. The application lifecycle, scene graph, script VM, rendering pipeline, audio system, and input handling.

- **[Dashboard Architecture](docs/xbox-architecture.md)** -- Technical reference for the dashboard's subsystems, and the things people commonly get wrong.

- **[XIP Archive Format](docs/xip-format.md)** -- The binary archive format. How XIP signatures worked, and the cat-and-mouse game where Microsoft kept moving the check and the community kept NOPing it out.

---

## The Alpha Builds

- **[Dashboard Evolution](docs/alpha-dashboard.md)** -- Every dashboard build from December 2000 through the final retail update. The 4-pod GAMES menu that was cut before launch. Hello Kitty demon skulls. Diablo II save icons. Developer high scores. Online leaderboard mockups from a year before Xbox Live.

- **[Cut Features](docs/cut-features.md)** -- Microsoft's game launcher with per-game news feeds, publisher metadata, custom menu extensions, and dedicated 3D art. The community rebuilt it over 20 years, mostly unaware of how close the original got before being cut.

---

## The Exploits

- **[XAP Scripting](docs/xap-scripting.md)** -- The VRML97 scripting language that defined the dashboard UI, and everything the community built on top of it:
  - The unsigned `.xbx` text file that enabled it all (the ghost of `games.xap`)
  - A game launcher built by hijacking the music player
  - A save game browser repurposed as an icon loader
  - An INI file writer turned into a database
  - A DVD player dongle check bypassed with one line
  - NT kernel device paths managed in JavaScript

- **[The Skin System](docs/skin-system.md)** -- From hex-patched colors to a texture preloader built on an invisible 3D mesh. How tHc, BSX, UIX, UIX Lite, and Theseus each solved the same problem differently.

---

## The Community

- **[UIX2: The Dashboard That Never Released](docs/uix2.md)** -- JbOnE's unreleased sequel. Game browser, disc installer, media player -- all inside Microsoft's engine. Killed by leaked builds.

- **[XboxDash.NeT](docs/xboxdash-net.md)** -- JbOnE's two-week sprint to build a dashboard from scratch. September 2004.

- **[Screenshots and Gallery](docs/gallery.md)** -- Alpha dashboards running on macOS in 2026. Broken early renders. Community skins. Xbox Live on Theseus.

---

## Related Projects

| Project | What |
|---------|------|
| **UIX Desktop** | Theseus engine running natively on macOS, Linux, and Windows via SDL2 and OpenGL |
| [UIX Lite](https://github.com/OfficialTeamUIX/UIX-Lite) | Patched 5960 dashboard with community XIPs and Rocky5's skinning tools |
| [Theseus](https://github.com/OfficialTeamUIX/Theseus) | Reverse-engineered dashboard engine for Xbox |

---

## TeamUIX

**Original**: acidbath, BobMcGee, Gcue, |ce, JbOnE, MPSnyper, tayior7, slay

**Today**: acidbath, ILTB, Mattie, Milenko, Odb718, ImOkRuOk, headph0ne, BigJx, Rocky5

The exploits documented here weren't figured out by one person. They accumulated over years across the Xbox modding scene -- tHc, BSX, UIX, and dozens of people sharing discoveries on forums and in IRC. UIX Lite pulled them together. This repo documents what they built.

**Special thanks to headph0ne** -- during COVID, he handed over a massive personal archive of UIX, UIX2, tHc, BSX, Commemorative builds, skins, and development files he'd been sitting on for almost two decades. A huge portion of the screenshots, build comparisons, and historical details in this repo wouldn't exist without that archive.

### Join Us

- **[TeamUIX Discord](https://discord.gg/qfVHTYD4xX)** -- Dashboard development, UIX Lite support, Theseus updates
- **[Xbox-Scene Discord](https://discord.gg/xbox-scene)** -- The Xbox modding community, reborn

---

This documentation was written with AI assistance. See **[AI_DISCLAIMER.md](AI_DISCLAIMER.md)** for details.

This repo contains documentation only -- no binaries, no decompiled source, no IDA/Ghidra projects, no Microsoft code. The Xbox, Xbox Dashboard, Direct3D, and related technologies are trademarks of Microsoft Corporation.
