# Installing LibreCarve (test builds)

Download the file for your computer from the **[Releases page](https://github.com/jhvanriper-rgb/LibreCarve-releases/releases)**.
These are early test builds: **always do an air cut first**, with the bit zeroed well above the
stock, and watch that the first move is straight up. LibreCarve writes G-code but never sends it to
a machine by itself.

## Linux

You need a 64-bit Intel or AMD PC with Linux Mint 21 or newer, Ubuntu 22.04 or newer, or another
distribution with glibc 2.35 or newer, and a graphics driver with Vulkan or OpenGL. Plan on about
400 MB of free disk space: the download is about 140 MB because it includes the depth model (99 MB,
Apache-2.0), so **Estimated depth** in Photo to Relief works offline with nothing more to download.

1. Download `librecarve-<version>-linux-x86_64.tar.gz` and `SHA256SUMS.txt`.
2. Optional: in the download folder run `sha256sum -c SHA256SUMS.txt`. It should print `OK`.
3. Unpack it and run the installer (no root password needed):

   ```
   tar -xzf librecarve-*-linux-x86_64.tar.gz
   cd librecarve-*-linux-x86_64
   ./install.sh
   ```

   This copies `librecarve` and `librecarve-cli` to `~/.local/bin`, the depth model to
   `~/.local/share/librecarve/bundled-models`, and adds LibreCarve to the applications menu with its icon. `PREFIX=/some/folder ./install.sh` installs somewhere else.
4. Start **LibreCarve** from the menu. If it does not show yet, log out and back in, or run
   `~/.local/bin/librecarve`. To try it without installing, run `./bin/librecarve` from the unpacked
   folder.

**Update:** unpack the newer release and run its `./install.sh`; it replaces the old files.
LibreCarve never updates itself: it does not look for new versions at startup or in the background,
and it downloads and installs nothing. **Help > Check for updates** looks at the Releases page once,
when you click it, and tells you if a newer version exists (it needs the `curl` program, which most
Linux systems have). **Remove:** run `./install.sh --uninstall` (it removes the depth model it
installed too). Your settings, tool database and logs stay in `~/.config/librecarve`; delete that folder too for a
clean slate.

## Windows

1. Download `librecarve-<version>-windows-x64.zip` and `SHA256SUMS-windows.txt`.
2. Unzip the folder anywhere you like. No installer is needed.
3. Double-click **librecarve.exe**.

The build is not code-signed, so Windows SmartScreen can say "Windows protected your PC". Click
**More info**, then **Run anyway**. To check your download, open PowerShell in the folder with the zip
and run `Get-FileHash .\librecarve-*-windows-x64.zip -Algorithm SHA256`. The result should match
`SHA256SUMS-windows.txt`. To remove it, delete the folder.

## Mac (Apple silicon and Intel, macOS 11 or newer)

1. Download `librecarve-<version>-macos-universal.zip` and `SHA256SUMS-macos.txt`.
2. Double-click the zip, then drag **LibreCarve.app** into Applications.
3. The first time, **right-click LibreCarve.app and choose Open**, then click **Open**. The app is not
   signed by an Apple developer account, so a plain double-click is blocked on the first launch.

If macOS says the app is "damaged", run this once in Terminal with the path where you put the app:
`xattr -dr com.apple.quarantine /Applications/LibreCarve.app`. On newer macOS you can also use
System Settings > Privacy & Security > **Open Anyway**. To check your download, run
`shasum -a 256 librecarve-*-macos-universal.zip` in the folder with the zip and compare it with
`SHA256SUMS-macos.txt`.

## Check the version

`librecarve --version` (or Help > About LibreCarve) prints something like
`librecarve 0.1.10 (1a2b3c4...)`. Put the whole line in every report.

## Big carves and memory

A large picture at a fine stepover can be millions of moves. LibreCarve checks how much memory your
computer has free and warns you, before the job gets that far, when the estimate is more than that.
If it warns, use a larger stepover, cut a smaller area, or split the job. Closing other programs
frees memory.

## Reporting a problem

Open an issue: [New issue](https://github.com/jhvanriper-rgb/LibreCarve-releases/issues/new/choose)
and pick **Bug report** or **Feature request**. The bug form asks for the version line above, your
system, numbered steps, what you expected and what happened, the log file if there is one, and the
drawing or G-code if you can share it. Please do not post files you are not allowed to share.

### The log file

LibreCarve writes a log while it runs, in `~/.config/librecarve/logs` on Linux (and the matching
settings folder on Windows and Mac):

| File | What it is |
| --- | --- |
| `librecarve.log` | the current (or last) run |
| `librecarve.1.log` | the run before that |
| `librecarve.2.log` | the one before that |

Each start moves the old log down one place. **After a crash, restart LibreCarve and attach
`librecarve.1.log`**: that is the run that crashed. If LibreCarve closes while you have unsaved
work, it offers to recover the drawing the next time it starts.

## Support

LibreCarve is free with no limits. If it saves you time, you can support its development on Patreon. The page is [patreon.com/c/librecarve](https://www.patreon.com/c/librecarve).
