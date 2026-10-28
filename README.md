# ai — local AI CLI launcher

A single command that lists the AI coding CLIs installed on this machine and lets
you pick one with the arrow keys, then hands the terminal over to it.

- Source of truth: `~/tools/ai-launcher/ai` (lives on `/mnt/d` via the `~/tools` symlink)
- Exposed on PATH as: `~/.local/bin/ai` -> symlink to the file above
- Dependencies: bash 4+, `stty`, coreutils. No `fzf` required.

## Usage

```
ai                     # interactive picker
ai claude              # skip the menu, launch by command name
ai claudecode          # ...or by lowercased label
ai qoder --resume      # everything after the tool name is passed through
ai -l                  # machine-readable list:  <cmd>\t<label>
ai -h                  # help
```

Keys in the picker: `↑`/`↓` or `k`/`j` to move, `1..9` to jump, `Home`/`End` for
first/last, `Enter` to launch, `q` or `Ctrl-C` to quit. Selection clamps at both
ends, and a bad tool name (`ai codex` when codex is missing) warns on stderr and
falls back to the menu instead of failing.

## Detected tools

`CANDIDATES` near the top of the script is the whole registry: `Label|command|extra args`.
A row only appears if `command -v <command>` succeeds, so installing a new CLI makes it
show up with no edit, and the extra-args column lets you pin a launch mode
(e.g. `"OpenCode|opencode|--thinking"`). The third field is word-split on purpose.

To add a tool that needs a label other than the command name, add one line to the array.
To support a new install location, nothing to do — PATH detection handles it.

## How the interactive part works

`menu()` draws the list once, then loops: read one byte -> update selection -> erase and
redraw in place. Erasing walks up `lines-1` rows with `\r\033[2K` and leaves the cursor on
the block's top line, which is why the block never drifts and never scrolls the terminal.
The block is printed without a trailing newline so `Enter` can emit one `\r\n` before exec.

## Terminal gotchas worth keeping in mind

These are the reasons the code looks the way it does. Re-read before "simplifying".

1. **`read -n1` returns an empty string for Enter.** CR and LF are line terminators, so the
   case pattern that works is `''` — `$'\r' | $'\n'` alone silently never matches and Enter
   just redraws.
2. **Arrow keys need raw mode.** `stty raw -echo -icanon` is what makes single-keypress
   handling work at all; without it the tty line-buffers until Enter.
3. **The `EXIT` trap does not run across `exec`.** The tty is restored *explicitly* by
   `restore_tty` inside `launch()` before exec'ing, because the launched TUI would otherwise
   inherit raw mode and leave your shell broken when it exits. Don't rely on the trap here.
4. **Truncate the path column, not the rendered line.** Slicing a string that already contains
   ANSI escapes corrupts the color state for the rest of the screen.
5. **Lone `ESC` vs. an escape sequence is ambiguous.** After `$'\x1b'` the next two bytes are
   read with `-t 1`; on timeout the key is ignored rather than treated as quit, so pressing
   Escape alone does nothing.
6. **Digits use `10#$key`.** Bare `$(( ))` on a digit string treats `08`/`09` as invalid octal.
7. **Non-tty falls back to plain output.** If stdout isn't a terminal, or `-l` was given, the
   script prints `cmd<TAB>label` and exits — this is what makes `ai -l | fzf` or shell
   completion scripts possible later.

## Testing without launching a real TUI

The `exec` line is the one thing you never want to trigger in tests. Stub it out and drive the
script through a real pty (`stty raw` needs a tty; a pipe alone won't exercise key handling):

```bash
sed 's|  exec "$cmd" "$@"|  echo "WOULD-EXEC: [$cmd] args=[$*]"; exit 0|' ai > /tmp/ait && chmod +x /tmp/ait

printf '%b' '\033[B\033[B\r' | timeout 6 script -qec 'COLUMNS=90 /tmp/ait' /dev/null \
  | sed 's/\x1b\[[0-9;?]*[A-Za-z]//g'          # two downs + Enter -> third tool
```

- Use `printf '%b'`, not `'%s'` — with `%s` the `\033` stays literal text and never becomes a keystroke.
- Send `\r` for Enter; raw mode leaves `icrnl` off.
- Always wrap in `timeout`: in raw mode the EOF character is disabled, so once the input bytes run
  out the script blocks forever instead of seeing end-of-input.
- To confirm the tty isn't left damaged: compare `stty -g` before and after a `q`-quit run.

## Reinstall on a new machine

```bash
chmod +x ~/tools/ai-launcher/ai
ln -sf ~/tools/ai-launcher/ai ~/.local/bin/ai
```

`~/.local/bin` must be on PATH (it is, via `~/.bashrc`). Note `~/.qoder/entry` sorts ahead of it,
so a future `ai` shipped by another tool would shadow this one — `type -a ai` to check.

Because the file lives on `/mnt/d` (drvfs), edit it from Linux. A Windows editor writing CRLF will
break the shebang and every `#!/...` line, and `stty`/`exec` behaviour on a Windows-mounted path is
worth re-testing after any change made from the Windows side.

## Ideas for later

- `fzf` backend when available (`command -v fzf`) — keep the raw-mode fallback for machines without it
- remember the last choice and default the highlight to it; `ai -` to relaunch the previous tool
- per-tool session flags worth exposing (all verified present): `claude --continue`/`--resume`,
  `qoder -c`/`--continue`/`--resume`, `opencode --continue`/`--session` (note: no `-c` here)
- show each tool's version or model in the third column (today it's the resolved path)
- read `CANDIDATES` from `~/.config/ai.conf` so new tools need no edit to the script
- shell completion for `ai <TAB>` from the existing `ai -l` output
- `ai -u`: check each tool for updates instead of launching
