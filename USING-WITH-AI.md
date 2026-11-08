# Using LibreCarve with an AI assistant

LibreCarve can be driven by an AI assistant such as Claude. You describe the job in plain words. The
assistant sets up the material, brings in the artwork, picks the bits, makes the toolpaths, checks
them and writes the G-code. It uses the same commands you could use yourself, and none of them opens a
dialog.

**The assistant never sends anything to your machine.** It makes the files and reports what it
found. You look at the report and the simulation, and you run the file yourself. Always do an air cut
first.

## What you need

- LibreCarve installed (see [INSTALL.md](INSTALL.md)). The download includes `librecarve-cli`, the
  small program that connects an assistant to LibreCarve. On Linux the installer puts it in
  `~/.local/bin` next to `librecarve`. On Windows and Mac it is in the download folder; use its full
  path in the settings below if it is not on your path.
- An assistant that supports MCP (the Model Context Protocol). Claude Desktop, Claude Code and many
  other tools do.

## Option 1: watch it work in the LibreCarve window

This is the best way to start. The assistant works in the window you have open, so you see the
drawing, toolpaths and simulation appear as it goes.

1. In LibreCarve choose **Window > Control channel...** and check **Control channel on**. The window
   shows "Listening on 127.0.0.1:7979". The channel is only reachable from your own computer, and it
   is off every time LibreCarve starts unless you check "Start automatically".
2. Tell your assistant about LibreCarve. For Claude Desktop, add this to its configuration file
   (`claude_desktop_config.json`), then restart Claude Desktop:

   ```
   { "mcpServers": { "librecarve": { "command": "librecarve-cli", "args": ["mcp", "--connect", "127.0.0.1:7979"] } } }
   ```

   For Claude Code, run this once in a terminal:

   ```
   claude mcp add librecarve -- librecarve-cli mcp --connect 127.0.0.1:7979
   ```

   If `librecarve-cli` is not on your path, use its full path in place of `librecarve-cli`.
3. You can open Claude and LibreCarve in either order. If LibreCarve is closed, the assistant waits
   for it for 30 seconds, then asks you to open it. It never starts a hidden second drawing.

## Option 2: no window

Leave out `--connect` and the assistant gets its own session inside the program, with no window:

```
{ "mcpServers": { "librecarve": { "command": "librecarve-cli", "args": ["mcp"] } } }
```

A session holds one job. If the connection ends, the job is gone unless the assistant saved it as a
project file, so ask it to save the job when it is done.

## Your first request

Start by telling the assistant to teach itself the steps. LibreCarve's help is available to it as
data. For example:

> Read the LibreCarve help page "ai-job-run", then follow it. I have a 3/4 inch thick piece of
> plywood, 12 by 24 inches. Cut the outline of the SVG file at `~/Pictures/sign.svg` with a 1/4 inch
> end mill. Check it, simulate it, and tell me everything you found before you write the G-code.

Good things to give it: the board size and thickness, which bits you own (it can read your tool
database), where your zero point is, and which machine you have. It will ask about anything it needs,
such as the thickness. You can also hand it a photo and say "carve this as a relief"; it uses Photo to
Relief with your bits.

## The training guide for your assistant

**[AGENT-GUIDE.md](AGENT-GUIDE.md)** is the full guide written for the assistant itself: every step of
a job, the commands and settings, and what to check after each. Give it to your assistant (attach the
file or paste its link) at the start of a session. The same material is inside LibreCarve under
Help > AI automation, and the assistant can read it as data with `help.page`.

## What the assistant reports

Before you cut, you should be given the part and material, each toolpath with its bit, depth, passes
and time, the check result and the simulation picture, every warning, the exact path of the G-code
file, and any number it guessed. If something is missing, ask for it. A failed check is a stop: the
assistant should not write G-code until the cause is fixed.

## The rules that keep it safe

- It can check a send to your machine but never perform one. The send, resume and continue buttons
  refuse clicks that come from the assistant.
- It does not decide that a job is safe to cut. You do.
- It does not invent feeds, speeds or depths. They come from your tool database or from you, and it
  says which numbers are guesses.
- It does not remove the lift or the pause that lets you start the router.
- Do not run a "computer use" assistant that moves your real mouse while the Send to Machine window is
  open. Such a program has the same reach as you at the keyboard.

The full rules are in the help page "AI safety and approval" (Help > AI automation).

## Other assistants and local models

Any program that can start `librecarve-cli mcp` as an MCP server works the same way. Local models
can drive it too if their tool supports MCP. Smaller models do better when you tell them to read the
help page `ai-job-run` first and to run one step at a time.

## Problems

If the assistant cannot connect, check that the control channel is on (Option 1), that the port number
in the settings matches the window, and that `librecarve-cli` runs when you type it in a terminal. To
report a problem, use [Issues](../../issues/new/choose) and include the version line from Help > About
LibreCarve.
