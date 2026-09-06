# OMP Keybindings Reference (`keybindings.yml`)

This reference documents the complete action IDs, default chords, syntax, and customization rules for `~/.omp/agent/keybindings.yml`.

## File Location & Structure

- Path: `~/.omp/agent/keybindings.yml`
- Format: Flat YAML mapping of `action.id: chord` or `action.id: [chord1, chord2]`
- To disable a chord: Set the action to `[]` (empty array).

```yaml
app.model.selectTemporary:
  - F12
  - Alt+P
app.model.select:
  - Alt+M
app.thinking.cycle:
  - Shift+Tab
app.editor.external: []
```

---

## Action IDs Catalog

### Models & Thinking
| Action ID | Default Chord | Description |
| --- | --- | --- |
| `app.model.cycleForward` | `Ctrl+P` | Cycle through configured `cycleOrder` models forward |
| `app.model.cycleBackward` | `Shift+Ctrl+P` | Cycle through `cycleOrder` models backward |
| `app.model.selectTemporary` | `Alt+P` | Pick a model temporarily for the current session only |
| `app.model.select` | `Alt+M` | Open interactive model selector and persist role assignments |
| `app.thinking.cycle` | `Shift+Tab` | Cycle reasoning effort levels (`minimal` → `low` → `medium` → `high` → `max`) |
| `app.thinking.toggle` | `Ctrl+T` | Toggle visibility of thinking/reasoning blocks |

### Session & Modes
| Action ID | Default Chord | Description |
| --- | --- | --- |
| `app.plan.toggle` | `Alt+Shift+P` | Toggle Plan Mode (`/plan`) |
| `app.history.search` | `Ctrl+R` | Interactive fuzzy search over prompt history |
| `app.agents.hub` | `Alt+A` | Open the Agent Hub dashboard (`/agents`) |
| `app.live.toggle` | `Ctrl+L` | Toggle live voice mode (`/live`) |
| `app.retry` | `Alt+R` | Retry the last assistant turn |

### Message Queue & Input
| Action ID | Default Chord | Description |
| --- | --- | --- |
| `app.message.followUp` | `Ctrl+Q`, `Ctrl+Enter` | Queue the typed draft as a follow-up steering message |
| `app.message.dequeue` | `Alt+Up`, `Shift+Up` | Pop the top queued message back into the prompt editor |
| `app.editor.external` | `Ctrl+G` | Open current prompt draft in `$VISUAL` / `$EDITOR` |

### Tool Activity & Display
| Action ID | Default Chord | Description |
| --- | --- | --- |
| `app.tools.expand` | `Ctrl+O` | Toggle expanding/collapsing all tool call outputs |
| `app.tools.toggleVisibility` | `Ctrl+Shift+O` | Toggle visibility of in-flight tool calls |
| `app.display.reset` | `Alt+L` | Redraw and clear terminal artifacts |

### Clipboard & Utilities
| Action ID | Default Chord | Description |
| --- | --- | --- |
| `app.clipboard.copyLine` | `Alt+Shift+L` | Copy current line to system clipboard |
| `app.clipboard.copyPrompt` | `Alt+Shift+C` | Copy entire prompt draft to clipboard |
| `app.clipboard.pasteImage` | `Ctrl+V` (Linux/macOS), `Alt+V` (Windows) | Paste clipboard image (falls back to text) |
| `app.clipboard.pasteTextRaw`| `Ctrl+Shift+V`, `Alt+Shift+V` | Paste text verbatim without newline collapse |
| `app.stt.toggle` | Unbound (Hold `Space`) | Toggle speech-to-text recording |

---

## Chord Notation Rules

- Modifiers: `Ctrl`, `Alt`, `Shift`, `Cmd` (or `Meta`), `Fn`
- Special keys: `Enter`, `Tab`, `Escape`, `Backspace`, `Delete`, `Up`, `Down`, `Left`, `Right`, `Home`, `End`, `PageUp`, `PageDown`, `Space`, `F1`-`F12`
- Case: Chord names are case-insensitive (`ctrl+p`, `Ctrl+P`, `CTRL+P` are identical).
- Multiple shortcuts: Map to a YAML list of chords.
