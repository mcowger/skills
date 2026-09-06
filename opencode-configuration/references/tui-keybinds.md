# OpenCode TUI, Themes & Keybinds Reference

TUI-specific settings live in `tui.json` / `tui.jsonc` (schema: `https://opencode.ai/tui.json`) — **not** `opencode.json`. Locations: `~/.config/opencode/tui.json` (global), `tui.json` in the project (walks up to git worktree), `.opencode/tui.json` in discovered config directories, or a custom file via `OPENCODE_TUI_CONFIG`. Source of truth: `packages/tui/src/config/index.tsx` and `keybind.ts`.

## tui.json Shape

```jsonc
{
  "$schema": "https://opencode.ai/tui.json",
  "theme": "neon-fuchsia",
  "leader_timeout": 2000,
  "keybinds": { "session_new": "f1" },
  "plugin": [["./plugins/my-tui-plugin.tsx", { "enabled": true }]],
  "plugin_enabled": { "some-plugin": true },
  "attention": { "enabled": true, "notifications": true, "sound": true, "volume": 0.4 },
  "prompt": { "max_height": 20, "max_width": "auto" },
  "scroll_speed": 3,
  "scroll_acceleration": { "enabled": true },
  "diff_style": "auto",
  "cursor": { "style": "block", "blinking": true },
  "mouse": true
}
```

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `theme` | string | built-in | Theme name — built-ins or a custom theme JSON in `themes/` |
| `keybinds` | object | — | Keybind overrides (see below) |
| `leader_timeout` | number | `2000` | ms to wait for the next key after the leader |
| `plugin` | array | — | TUI plugin specs (`"pkg"` or `["pkg", {options}]`); options may include `keybinds` |
| `plugin_enabled` | record | — | Enable/disable TUI plugins by name |
| `attention.enabled` | bool | `false` | Master switch for attention notifications + sounds |
| `attention.notifications` | bool | `true` | Desktop notifications |
| `attention.sound` | bool | `true` | Sounds |
| `attention.volume` | number | `0.4` | 0–1 |
| `attention.sound_pack` | string | `opencode.default` | Sound pack name |
| `attention.sounds` | record | — | Per-event sound paths: `default`, `question`, `permission`, `error`, `done`, `subagent_done` |
| `prompt.max_height` | number | — | Prompt textarea max height (lines) |
| `prompt.max_width` | number \| "auto" | — | Home prompt max width; `auto` scales with terminal |
| `scroll_speed` | number | — | ≥ 0.001 |
| `scroll_acceleration.enabled` | bool | — | Scroll acceleration |
| `diff_style` | string | — | `auto` (adapts to width) \| `stacked` (single column) |
| `cursor.style` | string | — | `block` \| `underline` \| `line` \| `default` (terminal setting preserved) |
| `cursor.blinking` | bool | — | No effect when style is `default` |
| `mouse` | bool | `true` | Mouse capture |

Legacy `theme`, `keybinds`, and `tui` keys inside `opencode.json` are deprecated and migrated automatically.

## Keybind syntax

```jsonc
{
  "keybinds": {
    "leader": "ctrl+x",
    "session_new": "f1",                          // single
    "session_delete": "ctrl+d,shift+d",            // multiple (comma-separated)
    "messages_copy": ["<leader>y", "ctrl+shift+c"],// array form
    "input_paste": { "key": "ctrl+v", "preventDefault": false }, // object form
    "session_compact": "none"                      // disabled ("none" or false)
  }
}
```

- `<leader>` references the leader key (default `ctrl+x`)
- Object form supports `key` (string or `{name, ctrl, shift, meta, super, hyper}`), `event` (`press`|`release`), `preventDefault`, `fallthrough`
- Unknown keybind names are dropped at load (the TUI logs a warning)
- Terminal caveat: some terminals don't send modifier keys with Enter by default (see Shift+Enter config for Windows Terminal in the docs)

## Default bindings (most useful)

### Leader-prefixed

| Action | Default | Description |
| --- | --- | --- |
| `leader` | `ctrl+x` | Leader key |
| `session_new` | `<leader>n` | New session |
| `session_list` | `<leader>l` | List sessions |
| `session_compact` | `<leader>c` | Compact the session |
| `session_export` | `<leader>x` | Export session to editor |
| `session_timeline` | `<leader>g` | Session timeline |
| `session_queued_prompts` | `<leader>q` | Manage queued prompts |
| `session_quick_switch_1..9` | `<leader>1..9` | Quick session slots |
| `model_list` | `<leader>m` | Model picker |
| `agent_list` | `<leader>a` | Agent list |
| `command_list` | `ctrl+p` | Command palette |
| `editor_open` | `<leader>e` | External editor |
| `theme_list` | `<leader>t` | Theme picker |
| `sidebar_toggle` | `<leader>b` | Toggle sidebar |
| `status_view` | `<leader>s` | View status |
| `messages_copy` | `<leader>y` | Copy message |
| `messages_undo` / `messages_redo` | `<leader>u` / `<leader>r` | Undo/redo message |
| `messages_toggle_conceal` / `tips_toggle` | `<leader>h` | Toggle concealment / tips |

### Non-leader

| Action | Default | Description |
| --- | --- | --- |
| `app_exit` | `ctrl+c,ctrl+d,<leader>q` | Exit |
| `session_interrupt` | `escape` | Interrupt current session |
| `session_rename` | `ctrl+r` | Rename session |
| `session_background` | `ctrl+b` | Background synchronous subagents |
| `agent_cycle` / `agent_cycle_reverse` | `tab` / `shift+tab` | Cycle agents |
| `variant_cycle` | `ctrl+t` | Cycle model variants |
| `model_cycle_recent` / `_reverse` | `f2` / `shift+f2` | Cycle recent models |
| `model_provider_list` | `ctrl+a` | Provider list from model dialog |
| `model_favorite_toggle` | `ctrl+f` | Favorite a model |
| `session_pin_toggle` | `ctrl+f` | Pin session in list |
| `session_child_first` | `<leader>down` | First child session |
| `session_child_cycle` / `_reverse` | `right` / `left` | Next/prev child session |
| `session_parent` | `up` | Parent session |
| `messages_page_up` / `messages_page_down` | `pageup,ctrl+alt+b` / `pagedown,ctrl+alt+f` | Page scrolling |
| `messages_half_page_up` / `messages_half_page_down` | `ctrl+alt+u` / `ctrl+alt+d` | Half-page scrolling |
| `messages_line_up` / `messages_line_down` | `ctrl+alt+y` / `ctrl+alt+e` | Line scrolling |
| `messages_first` / `messages_last` | `ctrl+g,home` / `ctrl+alt+g,end` | Jump to first/last message |
| `terminal_suspend` | `ctrl+z` | Suspend terminal |
| `which_key_toggle` | `ctrl+alt+k` | Which-key panel |
| `input_*` family | (readline-style) | Input editing: `ctrl+a`/`ctrl+e` line ends, `ctrl+k`/`ctrl+u` kill, `ctrl+w` word back, `alt+f`/`alt+b` word motion, etc. |
| `dialog.*`, `prompt.autocomplete.*` | (various) | Dialog navigation and autocomplete |

The full default map is printed in the docs at https://opencode.ai/docs/keybinds — any key present there can be overridden in `keybinds`.

## Themes

- Built-in themes switch via `<leader>t` (theme picker) or `"theme": "<name>"` in tui.json
- Custom themes are JSON files in `~/.config/opencode/themes/` (global) or `.opencode/themes/` (project); the file name (without `.json`) becomes the theme name
- Light/dark mode switching and mode locking: `theme_switch_mode` / `theme_mode_lock` keybinds (unbound by default)
