# OMP Keybindings Reference (`keybindings.yml`)

Complete action IDs, default chords, syntax, and customization rules for `~/.omp/agent/keybindings.yml`. Defaults come from `packages/coding-agent/src/config/keybindings.ts` and `packages/tui/src/keybindings.ts`; run `/hotkeys` in a session for the live list.

## File Location & Structure

- Path: `~/.omp/agent/keybindings.yml` (also `keybindings.yaml`; legacy `keybindings.json` is read and migrated on write-back).
- Format: flat YAML mapping of `action.id: chord` or `action.id: [chord1, chord2]`.
- Disable a chord: set the action to `[]`.
- With a named profile, the default profile's `keybindings.yml` loads first and the active profile's file overrides action by action.

```yaml
app.model.selectTemporary:
  - F12
  - Alt+P
app.thinking.cycle:
  - Shift+Tab
app.editor.external: []
```

Chord names are case-insensitive (`ctrl+p`, `Ctrl+P`, `CTRL+P` are identical).

---

## App Action IDs

### Models & Thinking
| Action ID | Default | Description |
| --- | --- | --- |
| `app.model.cycleForward` | `Ctrl+P` | Cycle role models forward |
| `app.model.cycleBackward` | `Shift+Ctrl+P` | Cycle role models backward |
| `app.model.selectTemporary` | `Alt+P` | Pick a model temporarily for the current session |
| `app.model.select` | `Alt+M` | Open the model selector and set role assignments |
| `app.thinking.cycle` | `Shift+Tab` | Cycle reasoning effort level |
| `app.thinking.toggle` | `Ctrl+T` | Toggle thinking-block visibility |

### Session, Modes & Interrupts
| Action ID | Default | Description |
| --- | --- | --- |
| `app.interrupt` | `Escape` | Interrupt the current operation |
| `app.clear` | `Ctrl+C` | Clear the draft / cancel |
| `app.exit` | `Ctrl+D` | Exit the application |
| `app.suspend` | `Ctrl+Z` | Suspend the application |
| `app.plan.toggle` | `Alt+Shift+P` | Toggle plan mode (`/plan`) |
| `app.retry` | `F5`, `Alt+R` | Retry the last failed assistant turn |
| `app.session.new` | _unbound_ | Create a new session |
| `app.session.tree` | _unbound_ | Show the session tree |
| `app.session.fork` | _unbound_ | Fork the session |
| `app.session.resume` | _unbound_ | Resume a session |
| `app.session.observe` | `Ctrl+S` | Open the session observer |
| `app.session.rename` | `Ctrl+R` | Rename the current session |
| `app.session.delete` | `Ctrl+D` | Delete the session |
| `app.session.deleteNoninvasive` | `Ctrl+Backspace` | Delete the session non-invasively |
| `app.session.togglePath` | `Ctrl+P` | Toggle session path display |
| `app.session.toggleSort` | `Ctrl+S` | Toggle session sort order |
| `app.tree.foldOrUp` | `Ctrl+Left`, `Alt+Left` | Fold or move up in the tree |
| `app.tree.unfoldOrDown` | `Ctrl+Right`, `Alt+Right` | Unfold or move down in the tree |
| `app.history.search` | `Ctrl+R` | Fuzzy-search prompt history |
| `app.agents.hub` | `Alt+A` | Open the Agent Hub (`/agents`) |
| `app.live.toggle` | `Ctrl+L` | Start/stop live voice mode (`/live`) |

### Message Queue & Input
| Action ID | Default | Description |
| --- | --- | --- |
| `app.message.followUp` | `Ctrl+Q`, `Ctrl+Enter` | Queue the draft as a follow-up message |
| `app.message.dequeue` | `Alt+Up`, `Shift+Up` | Pop the top queued message back into the editor |
| `app.editor.external` | `Ctrl+G` | Edit the draft in `$VISUAL` / `$EDITOR` |

### Tool Activity & Display
| Action ID | Default | Description |
| --- | --- | --- |
| `app.tools.expand` | `Ctrl+O` | Toggle expanding/collapsing tool output |
| `app.tools.toggleVisibility` | `Ctrl+Shift+O` | Show/hide in-flight tool activity |
| `app.display.reset` | `Alt+L` | Redraw and clear terminal artifacts |

### Clipboard, Voice & Utilities
| Action ID | Default | Description |
| --- | --- | --- |
| `app.clipboard.copyLine` | `Alt+Shift+L` | Copy the current line |
| `app.clipboard.copyPrompt` | `Alt+Shift+C` | Copy the whole prompt |
| `app.clipboard.pasteImage` | Linux: `Ctrl+V`; macOS: `Ctrl+V`, `Cmd+V`; Windows: `Ctrl+V`, `Alt+V` | Paste clipboard image (text fallback) |
| `app.clipboard.pasteTextRaw` | `Ctrl+Shift+V`, `Alt+Shift+V` | Paste text verbatim (no newline collapse) |
| `app.stt.toggle` | _unbound_ (hold `Space`) | Toggle speech-to-text (push-to-talk by default) |

---

## TUI Action IDs

The TUI editor, input, and select actions are namespaced under `tui.*` and can be remapped the same way.

| Group | Action IDs |
| --- | --- |
| Editor navigation | `tui.editor.cursorUp`, `cursorDown`, `cursorLeft`, `cursorRight`, `cursorWordLeft`, `cursorWordRight`, `cursorLineStart`, `cursorLineEnd`, `jumpForward`, `jumpBackward`, `pageUp`, `pageDown` |
| Editor editing | `tui.editor.deleteCharBackward`, `deleteCharForward`, `deleteWordBackward`, `deleteWordForward`, `deleteToLineStart`, `deleteToLineEnd`, `yank`, `yankPop`, `undo`, `spellingSuggestions` |
| Input | `tui.input.newLine`, `tui.input.submit`, `tui.input.tab`, `tui.input.copy` |
| Select list | `tui.select.up`, `tui.select.down`, `tui.select.pageUp`, `tui.select.pageDown`, `tui.select.confirm`, `tui.select.cancel` |

---

## Vim Editing Mode

Off by default. Enable with **Vim Editing Mode** in `/settings` (Interaction → Input) or `tui.vimMode: true`.

- The prompt starts in Insert mode; `Escape` switches to Normal.
- Normal mode: `hjkl`, `0`/`^`/`$`, `w`/`b`/`e`, `gg`/`G`, counts, `x`/`D`/`C`, `dd`/`yy`/`cc`, `p`/`P`, `u`.
- Operators take motions or text objects (`diw`, `ca(`, `ci"`, `dap`).
- `v`/`V` start a Visual / Visual-line selection; `y` copies, `d` deletes, `c` changes.
- `tui.vimModeDisplay` controls the status-line indicator: `text` (default), `icon`, or `none`.
- `Escape` is only consumed when Vim mode has something to do; otherwise it falls through to `app.interrupt`. App chords (`Ctrl` combos, `Enter`, `Tab`) keep their normal behavior in every mode.

---

## Recover a Cleared Draft

Press `Ctrl+C` to clear an unsent draft, then `Up` to recall it (controlled by `composer.recallClearedDrafts`, default `true`). Drafts remain editable and are kept in the editor's bounded history (100 entries); they are not written to persistent history and do not appear in `Ctrl+R` search.

---

## Chord Notation Rules

- Modifiers: `Ctrl`, `Alt`, `Shift`, `Cmd`/`Meta`, `Super`
- Special keys: `Enter`, `Tab`, `Escape`, `Backspace`, `Delete`, `Up`, `Down`, `Left`, `Right`, `Home`, `End`, `PageUp`, `PageDown`, `Space`, `F1`–`F12`
- Multiple shortcuts: map to a YAML list of chords.
- Older unqualified names (e.g. `selectModel`, `retry`, `cursorUp`) are migrated to namespaced IDs at load time, but new configs should use the namespaced IDs above.

### Platform caveats

- Windows Terminal does not emit a distinct `Ctrl+Enter`, so `app.message.followUp` also binds `Ctrl+Q`.
- `F5` leads `app.retry` because every terminal delivers it verbatim; modified Enter chords are swallowed by some terminals.
- macOS Terminal.app consumes Option for character composition, so `app.message.dequeue` also binds `Shift+Up`.
- Windows Terminal may intercept `Ctrl+V`; the `Alt+V` fallback handles clipboard image paste there.
