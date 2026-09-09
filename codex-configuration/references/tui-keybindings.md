# Codex TUI, Themes & Keybindings Reference

Configuration for `[tui]` and `[tui.keymap.*]` in `~/.codex/config.toml`. Source of truth: `Tui` / `TuiKeymap` in `codex-rs/config/src/types.rs` + `tui_keymap.rs` (unknown fields rejected; key specs canonicalized at parse time).

## Display & behavior

```toml
[tui]
theme = "dark"                       # /theme to switch; custom themes in $CODEX_HOME/themes
status_line = ["model-with-reasoning", "current-dir", "thread-name"]  # default set when unset
status_line_use_colors = true
terminal_title = ["activity", "thread-name", "project-name"]          # default set when unset
alternate_screen = "auto"            # auto (default) | always | never (also --no-alt-screen)
animations = true
whimsy = true                        # decorative effects (needs animations)
show_tooltips = true
show_server_version_notice = true
auto_recap = true                    # background recaps when unfocused (/recap on demand)
vim_mode_default = false
question_esc_back = true             # Esc returns from async questions, keeps draft
raw_output_mode = false              # copy-friendly scrollback transcript
session_picker_view = "dense"        # dense (default) | comfortable (profiles: same enum)
resume_cwd = "session"               # current | session (unset = prompt when they differ)
disable_paste_burst = false          # burst-paste detection off (legacy top-level key also works)
terminal_resize_reflow_max_rows = 1000  # resize replay cap; 0 = keep all; unset = terminal default
pet = "my-pet"                       # $CODEX_HOME/pets/<id>/pet.json
pet_anchor = "composer"              # composer (default) | screen-bottom
```

Notifications (`[tui]` flattened): `notifications` (on/off), `notification_method` (auto default), `notification_condition` (`unfocused` default — notify only when unfocused — or always).
Related top-level: `file_opener` (URI-scheme opener for citations, e.g. `"vscode"`), `hide_agent_reasoning` (default false), `show_raw_agent_reasoning` (default false), `[history] persistence = "save-all" | "none"`, `max_bytes`.

## Keybindings (`[tui.keymap.<context>]`)

```toml
[tui.keymap.chat]
previous_permission_mode = "ctrl-p"
next_permission_mode = "ctrl-n"

[tui.keymap.global]
open_transcript = ["ctrl-t", "ctrl-x ctrl-t"]
copy = []                            # empty list = explicitly unbound (no fallthrough)
```

- **Contexts**: `global`, `chat`, `composer`, `editor`, `vim_normal`, `vim_operator`, `vim_search`, `vim_text_object`, `pager`, `list`, `agents`, `approval`. Context bindings win over `global`; selected chat/composer actions fall back through `global` at runtime.
- **Value shape**: one key/chord string, or a list = alternatives. `[]` unbinds (and blocks fallthrough to defaults — an explicit empty list is still a configured value).
- **Spec syntax**: modifiers `ctrl-`/`alt-`/`shift-`/`super-` (canonical order `ctrl-alt-shift-<key>`), one optional second stroke separated by a space (`"ctrl-x ctrl-s"`, max 2 strokes), function keys `f1`–`f24`, aliases (`escape`→`esc`, `pageup`→`page-up`). Malformed specs fail config load with a diagnostic.
- Unknown actions/fields are rejected — check the tables below for exact names.
- **Project layers cannot remap `tui.keymap.chat.previous_permission_mode` / `next_permission_mode`** (sanitized — permission escalation must not come from repo contents).

### Action catalog

**global** — `open_agents`, `open_transcript`, `open_external_editor`, `copy`, `clear_terminal`, `submit`, `queue`, `toggle_shortcuts`, `toggle_vim_mode`, `toggle_fast_mode`, `toggle_raw_output`, `toggle_side_conversation`.

**chat** — `toggle_voice_mute`, `interrupt_turn`, `decrease_reasoning_effort`, `increase_reasoning_effort`, `previous_permission_mode`, `next_permission_mode`, `edit_queued_message`, `prompt_stack_back`, `skip_question`.

**composer** — `submit`, `queue`, `toggle_shortcuts`, `history_search_previous`, `history_search_next`.

**editor** — cursor/insert/delete family: `insert_newline`, `move_left/right/up/down`, `move_word_left/right`, `move_line_start/end`, `delete_backward/forward`, `delete_backward_word`, `delete_forward_word`, `kill_line_start/whole_line/end`, `yank`.

**vim_normal** — `enter_insert`, `append_after_cursor`, `append_line_end`, `insert_line_start`, `open_line_below/above`, `enter_replace_mode`, motions (`move_left/right/up/down`, `move_word_forward/backward/end`, `move_line_start/end`, `find_forward/backward`, `till_forward/backward`, `jump_top/bottom`), edits (`delete_char`, `replace_char`, `repeat_last_change`, `substitute_char`, `delete_to_line_end`, `change_to_line_end`, `yank_line`, `paste_after`), operators (`start_delete/yank/change_operator`), `undo`, `redo`, `cancel_operator`.

**vim_operator** — operator-pending motions: `delete_line`, `yank_line`, `motion_left/right/up/down`, `motion_word_forward/backward/end`, `motion_line_start/end`, `motion_find_forward/backward`, `motion_till_forward/backward`, `motion_jump_top/bottom`, `select_inner/around_text_object`, `cancel`.

**vim_search** — `forward`, `backward`, `next`, `previous`.

**vim_text_object** — `word`, `big_word`, `parentheses`, `brackets`, `braces`, `double_quote`, `single_quote`, `backtick`, `cancel`.

**pager** — `scroll_up/down`, `page_up/down`, `half_page_up/down`, `jump_top/bottom`, `close`, `close_transcript`.

**list** — `move_up/down/left/right`, `page_up/down`, `jump_top/bottom`, `accept`, `cancel`.

**agents** — `resume`, `search`, `new_task`, `rename`, `stop`, `toggle_grouping`.

**approval** — `open_fullscreen`, `open_thread`, `approve`, `approve_for_session`, `approve_for_prefix`, `deny`, `decline`, `cancel` (elicitation).
