# ai — local AI CLI launcher

One command that detects the AI coding CLIs installed on your machine, shows them in an
arrow-key picker, and hands the terminal over to the one you choose.

Single bash file, no `fzf`, no network access, no dependencies beyond a POSIX-ish userland —
`stty` for the picker and `stat` for the launch check are the only optional lookups.

## Requirements

- bash 4+ (associative arrays; the script checks and says so if you are on bash 3.2)
- `stty` — required for the arrow-key picker only; `ai <tool>`, `-l`, `-a` and `doctor` work without it
- `tput` — optional; without it the picker assumes an 80x24 terminal
- `stat` — optional; without it the launch-time permission check is skipped

Only GNU or BSD coreutils are assumed: paths are canonicalised without `readlink -f`, and file
modes are read with `stat -c` falling back to `stat -f`. `ai doctor` reports what it found.

## Install

```bash
git clone <your-fork-or-mirror> ai-launcher
cd ai-launcher

chmod +x ai          # only needed for a fresh clone, or if the exec bit was lost
./ai install         # links ai into ~/.local/bin
ai                   # works from any shell, in any directory
```

`install` accepts a target override and a copy mode:

```bash
AI_BIN_DIR=/usr/local/bin ./ai install     # system-wide (needs write access)
./ai install --copy                        # copy instead of symlink
./ai uninstall                             # remove it, only if it points back here
```

Notes:

- Symlink is the default: edit `ai` in the clone and every shell picks up the change.
  `--copy` is for when the clone will move or disappear.
- If the target directory is not on `PATH`, `install` prints the `export PATH=...` line to add.
- `install` refuses to overwrite a file at the target that is not this script — nothing you
  installed by hand gets clobbered, and `uninstall` refuses to delete a foreign `ai`.
- Run `./ai install` from a shell where `hash -r` has not been done? `type -a ai` shows which
  copy wins if another tool already ships an `ai`.

## Usage

```
ai                     interactive picker
ai claude              launch by command name, no menu
ai claudecode          ...or by lowercased label
ai qoder --resume      everything after the tool name passes through to the tool
ai -l                  machine-readable list:  <cmd>\t<label>
ai -a, --all           registry + discovered tools, found and missing
ai --missing           registry entries not installed yet (with install hints)
ai doctor              dependency, discovery, registry and trust check
ai help                full usage
ai --version           the version
```

Picker keys: `↑`/`↓` or `j`/`k` move, `1-9` jump, `g`/`G` or `Home`/`End` first/last,
`Enter` launch, `q` or `Ctrl-C` quit.

An unknown name shows the menu interactively, and exits 1 when there is no terminal to show it
to — `ai codex || npm i -g @openai/codex` in a script distinguishes a typo from success.
The selection clamps at both ends, and on a short terminal the list windows with `▾ N more`.

## How detection works

Three layers, merged and deduplicated by command name:

1. **Curated registry** (`CANDIDATES` near the top of the script). Provides friendly labels and
   install hints. Missing tools still appear in `ai -a` / `ai --missing`, just not in the picker.
2. **Generic discovery.** Every executable in every `PATH` directory plus a handful of vendor
   install dirs (`~/.local/bin`, `~/.bun/bin`, `~/.cargo/bin`, `~/go/bin`, `/usr/local/bin`,
   `~/.linuxbrew/bin`, ...) is matched against agent-name patterns. A CLI you install with any
   package manager shows up with no edit to this script.
   - A name matches when a known token appears at a **separator boundary** — the start, the end,
     or a whole hyphen/underscore/dot component. That finds `claude`, `codebuff` and `freebuff`,
     plus future names like `free-claude-code`, while `buffer` and `sample` stay out. The token
     list is `AGENT_TOKENS` and is extendable through the environment.
   - Bare `something-agent` names need an AI-ish prefix, otherwise `ssh-agent`, `gpg-agent` and
     `systemd-*‑agent` would show up as coding tools. There is a deny list for editors, wrappers
     and daemon/helper/shim-style names too.
   - Directories under `/mnt` are skipped, so Windows shares on WSL do not stall the scan.
   - Entries are keyed by the binary's base name, so the same tool reached from two directories
     — or named as a full path in your config — produces one row, not several.
3. **Your config file.** See below.

### Config

Default `~/.config/ai/tools.conf` (override with `AI_CONFIG`). One entry per line:

```
# add a tool that needs a specific launch mode
My Agent|my-agent|--model fast
# an absolute path works too, for tools that are not on PATH
Vendor|/opt/vendor/bin/agent-cli
# hide something discovery keeps finding (bare name, or the same base name as a path entry)
!some-unrelated-binary
```

`Label|command|args` — the args field is split on spaces only; it is deliberately *not*
glob-expanded, so `--mode *` reaches the tool as the literal `*`. `!command` hides that command
from both the registry and discovery, matched on the binary's base name. Comments must start the
line, because a path or argument may legitimately contain `#`.

### Environment knobs

| Variable | Effect |
| --- | --- |
| `AI_CONFIG` | path to the config file |
| `AI_BIN_DIR` | install target directory (default `~/.local/bin`) |
| `AI_DISCOVER=0` | registry only, skip the filesystem scan |
| `AI_EXTRA_DIRS="a:b"` | additional directories to scan |
| `AI_AGENT_TOKENS='claude\|mytool'` | extend the name patterns treated as agents |
| `AI_TRUST_INSECURE=1` | allow launching a tool the permission check would refuse |

## Security model

The picker execs binaries, so `ai` applies the same rule every time, just before the `exec`:

- refuse if the file is writable by accounts other than its owner (world-writable),
- refuse if its directory is,
- refuse if the file is owned by anyone other than root or you,
- skip `PATH` directories under `/mnt` during the scan.

`ai doctor` lists anything it would refuse, and `AI_TRUST_INSECURE=1` turns the refusal off for a
deliberate case. The check follows symlinks (`stat -L`) because npm-style global installs are
symlinks whose own mode is always 777 — that would otherwise flag every tool you actually use.

Group-writable files are allowed: shared team installs are common and the group is usually your
own. Discovery does search some directories that are not on your `PATH` (the vendor install dirs),
so the checks above matter more here than for a plain shell lookup.

## Registry

npm/pip package names below were checked against the public registries; an empty hint means the
tool normally arrives through its own installer or an app bundle.

| Label | Command | Install hint |
| --- | --- | --- |
| Qoder | `qoder` | vendor installer |
| OpenCode | `opencode` | `npm:opencode-ai` |
| Claude Code | `claude` | `npm:@anthropic-ai/claude-code` |
| Codex | `codex` | `npm:@openai/codex` |
| Gemini CLI | `gemini` | `npm:@google/gemini-cli` |
| Copilot CLI | `copilot` | `npm:@github/copilot` |
| Cursor Agent | `cursor-agent` | vendor installer |
| Amp | `amp` | `npm:@sourcegraph/amp` |
| Droid | `droid` | `npm:@factory/cli` |
| Kiro | `kiro-cli` | vendor installer |
| Augment | `augment` | vendor installer |
| Goose | `goose` | vendor installer |
| Crush | `crush` | vendor installer |
| Aider | `aider` | `pipx:aider-chat` |
| Kilo | `kilo` | `npm:@kilocode/cli` |
| Cline | `clite` | `npm:@cline/cli` |
| OpenHands | `openhands` | vendor installer |
| Continue | `cn` | `npm:@continuedev/cli` |
| Qwen Code | `qwen` | `npm:@qwen-code/qwen-code` |
| Kimi Code | `kimi` | `npm:@moonshot-ai/kimi-code` |
| iFlow | `iflow` | `npm:@iflow-ai/iflow-cli` |
| Codebuff | `codebuff` | `npm:codebuff` |
| Freebuff | `freebuff` | `npm:freebuff` |

Some vendors ship a binary whose name is not the product name (`cline` → `clite`,
`continue` → `cn`). That is why the registry has a command column and discovery matches names,
not labels.

## Implementation notes

`menu()` draws the list once, then loops: read one key → update selection → erase and redraw in
place. Erasing walks up `lines-1` rows with `\r\033[2K` and leaves the cursor on the block's top
line, which is why the block never drifts and never scrolls your terminal. The block is printed
without a trailing newline so `Enter` can emit one `\r\n` before exec.

These are the reasons the code looks the way it does. Re-read before "simplifying":

1. **`read -n1` returns an empty string for Enter.** CR and LF are line terminators, so the case
   pattern that works is `''` — `$'\r' | $'\n'` alone silently never matches and Enter just
   redraws.
2. **Arrow keys need raw mode.** `stty raw -echo` is what makes single-keypress handling work at
   all; without it the tty line-buffers until Enter.
3. **The `EXIT` trap does not run across `exec`.** The tty is restored *explicitly* by
   `restore_tty()` inside `launch()` before exec'ing, because the launched TUI would otherwise
   inherit raw mode and leave your shell broken when it exits. Do not rely on the trap here. The
   trap *is* installed before `stty raw -echo`, so a signal in between cannot strand the terminal.
4. **Truncate the path column, never the rendered line.** Slicing a string that already contains
   ANSI escapes corrupts the color state for the rest of the screen.
5. **Lone `ESC` vs. an escape sequence is ambiguous.** After `$'\x1b'` the next two bytes are read
   with `-t 1`; on timeout the key is ignored rather than treated as quit, so pressing Escape
   alone does nothing.
6. **Digits use `10#$key`.** Bare `$(( ))` on a digit string treats `08`/`09` as invalid octal.
7. **Non-tty falls back to plain output.** If stdout is not a terminal, or `-l` was given, the
   script prints `cmd<TAB>label` and exits — this is what makes `ai -l | fzf` and shell completion
   possible later.
8. **Startup is fork-free on purpose.** Detection uses `-f`/`-x` tests and builtin regex instead of
   `$(command -v)` and `grep` pipelines, because a process per lookup made startup ~1.6s on WSL
   versus ~0.15s now. Keep new code fork-free — including helpers, which is why `short_path` and
   `add_entry` set globals instead of printing.
9. **The regex needs a cheap guard in front of it.** `[[ $base == *@(${AGENT_TOKENS})* ]]` rejects
   every unrelated filename in `/usr/bin` before the wider `AGENT_RE` runs; without it that regex
   alone cost ~0.25s per invocation, doubling startup.
10. **`stat` without `-L` reports the symlink, not the tool.** A symlink's own mode is always 777,
    so the permission check read every npm-global install as world-writable until `-L` was added.
11. **No `set -e`.** Failures are handled where they happen — a global `-e` would exit the picker
    on the first non-matching `[[ ]]` test, which is how `[ ... ] && continue` patterns are used
    throughout the loops.
12. **Empty arrays and `set -u` do not mix before bash 4.4.** Expansions that can legitimately be
    empty — `tools`, `all`, `DIRS`, `extra`, the doctor notes — are written `${a[@]+"${a[@]}"}`.
    The bash 4 version guard at the top of the script covers the rest (`declare -A`, `${s^}`).

## Testing without launching a real TUI

The `exec` line is the one thing you never want to trigger in tests. Stub it out and drive the
script through a real pty (`stty raw` needs a tty; a pipe alone will not exercise key handling):

```bash
sed -E 's|^([[:space:]]*)exec .*$|\1echo "WOULD-EXEC: pre=[${extra[*]:-}] args=[$*]"; exit 0|' ai > /tmp/ait && chmod +x /tmp/ait

printf '%b' '\033[B\033[B\r' | timeout 6 script -qec 'COLUMNS=90 /tmp/ait' /dev/null \
  | sed 's/\x1b\[[0-9;?]*[A-Za-z]//g'          # two downs + Enter -> third tool
```

- Use `printf '%b'`, not `'%s'` — with `%s` the `\033` stays literal text and never becomes a keystroke.
- Send `\r` for Enter; raw mode leaves `icrnl` off.
- Always wrap in `timeout`: in raw mode the EOF character is disabled, so once the input bytes run
  out the script blocks forever instead of seeing end-of-input.
- To confirm the tty is not left damaged, compare `stty -g` before and after a `q`-quit run.
- Prefer fake tools in a scratch directory plus `AI_EXTRA_DIRS` over installing anything real.

## Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| `bash 4 or newer is required` | You are on a stock macOS bash 3.2. Install bash (`brew install bash`) or another package manager's build. |
| `ai: "stty" is required for arrow-key navigation` | Minimal container image. `stty` is only needed for the picker; `ai <tool>` and `ai -l` work without it. |
| `ai: refusing to launch /path/to/tool` | The file or its directory is world-writable, or the file is owned by another account. Fix the mode/ownership, or set `AI_TRUST_INSECURE=1` if it is deliberate; `ai doctor` lists every such tool. |
| A tool you know is installed does not appear | It is not in a scanned directory (`ai doctor` prints the count) or its name has no known token on a separator boundary. Add it to the config file, pass its directory via `AI_EXTRA_DIRS`, or extend `AI_AGENT_TOKENS`. |
| Two tools with the same binary name, one row | Intended: entries are keyed by base name and the first directory in `PATH` order wins. Name the one you want in the config file. |
| A non-AI tool appears | `!command` it in the config file, or narrow patterns with `AI_AGENT_TOKENS`. Please also open an issue — the deny list belongs in the script. |
| Slow startup | A scanned directory is a network mount. Use `AI_DISCOVER=0` or trim `PATH`/`AI_EXTRA_DIRS`. |
| Terminal garbled after quitting | Should not happen; check `stty -g` and file a report with your terminal and `$TERM`. |
| `command not found: ai` after install | The bin dir is not on `PATH`; `install` prints the export line to add to your shell rc. |
| Shebang breaks on a fresh checkout | The file was edited with a Windows editor that wrote CRLF. Restore LF endings and the exec bit (`chmod +x ai`). |

## Ideas for later

- `fzf` backend when available, keeping the raw-mode fallback for machines without it
- remember the last choice and default the highlight to it; `ai -` to relaunch the previous tool
- per-tool session flags worth exposing (verified present): `claude --continue`/`--resume`,
  `qoder -c`/`--continue`/`--resume`, `opencode --continue`/`--session` (no `-c` there)
- show each tool's version or model in the third column (today it is the resolved path)
- shell completion for `ai <TAB>` from the existing `ai -l` output
- `ai -u`: check each tool for updates instead of launching it
- keep the registry in a data file so new tools need no code change
