# LibreCarve

**LibreCarve** is a CNC design and carving program for hobby and small-shop routers. You draw or
import artwork, set up the material, pick bits and feeds, make toolpaths, simulate the cut, check it
for gouges, and save G-code. It has no edition limits: no work-area cap, no limit on imported 3D
models, and no features held back for a higher tier.

This repository holds the **downloads and the bug tracker**. It does not contain source code.

## Download

Get the newest test build from the **[Releases page](../../releases)**.

| Computer | File |
| --- | --- |
| Linux (64-bit Intel or AMD) | `librecarve-<version>-linux-x86_64.tar.gz` |
| Windows (64-bit) | `librecarve-<version>-windows-x64.zip` |
| Mac (Apple silicon and Intel) | `librecarve-<version>-macos-universal.zip` |

The Releases page lists what changed in each version under **What's new**. Every file has a checksum list on the same page (`SHA256SUMS*.txt`) so you can confirm your download.
Step-by-step instructions for each system are in **[INSTALL.md](INSTALL.md)**.

## Status: early test builds

These are test builds, not a finished product.

- **Always do an air cut first**, with the bit zeroed well above the stock, and watch that the first
  move is straight up.
- LibreCarve writes G-code but never sends it to a machine by itself. You review it and run it.
- The main target machine so far is the Onefinity Woodworker. Other machines may need their settings
  checked in Toolpaths > Machine Setup.
- Windows and Mac builds are newer than the Linux build, so expect rough edges there.
- LibreCarve is in beta, so every release is marked **Pre-release** on the Releases page.

## What it does

- Draw shapes, import SVG, DXF and other vector files, or trace a picture into outlines.
- Set up the material, the stock size and the zero point, with a picture of the stock on screen.
- Toolpaths: profile, pocket, drill, V-carve (including stacked V-carve), engraving, chamfer,
  countersink, thread milling, nesting, and 3D relief from a picture or a 3D model.
- **Photo to Relief:** turn a photo into a carving. Heights can come from the brightness of the
  picture or from an estimated depth map (a depth model comes with the download and works offline).
  Choose the rough bit and finish bit from your tool database, including tapered ball nose bits, set
  how deep the relief goes as a percent of the board, push the background to the floor, bring back
  fine detail from the photo, and turn the 3D preview before you make the job.
- **Quick Start:** a guided walk through material setup, adding a design, sizing it to the board and
  picking tools, for 2D drawings, 3D models and pictures.
- Rulers along the drawing view, a tool database for your own bits, and progress with a Cancel button
  for long jobs. Large carves are not capped at a fixed size: LibreCarve estimates the size of a
  relief toolpath, warns when it is more than your free memory, and refuses only what truly cannot
  fit.
- Simulate the cut in 2D or 3D and check every toolpath for gouges before you save G-code.
- G-code for the Onefinity Woodworker and several other controllers.
- Drive it from an AI assistant through its command interface (optional).

## Support

LibreCarve is free with no limits. If it saves you time, you can support its development on Patreon. The page is [patreon.com/c/librecarve](https://www.patreon.com/c/librecarve). Nothing in LibreCarve depends on it.

## Report a problem or ask for a feature

Use **[Issues](../../issues/new/choose)** and pick **Bug report** or **Feature request**. A good bug
report has the version (`librecarve --version` or Help > About LibreCarve), your system, numbered
steps, and the log file. INSTALL.md says where the log file is.

## Credits and licenses

LibreCarve is built on [CadCraft](https://github.com/storytold/cadcraft), a drafting program by the
ArtCraft team and contributors, which is licensed under MIT OR Apache-2.0. The license files and the
notice that ship with the program are in this repository (`LICENSE-MIT`, `LICENSE-APACHE`, `NOTICE`)
and inside every download, along with the list of bundled assets. App icon designed by James Van Riper.
LibreCarve is not affiliated with any other software company.
