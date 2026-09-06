# Pi Keybindings Reference

Global-only file: `~/.pi/agent/keybindings.json`. Run `/reload` after editing;
inspect effective bindings with `/hotkeys`. Legacy unnamespaced IDs (e.g. `cursorUp`)
are migrated automatically.

```json
{
  "tui.editor.historyPrevious": "ctrl+p",
  "tui.editor.deleteWordBackward": ["ctrl+w", "alt+backspace"]
}
```

Each action takes one key or an array. Key syntax: `ctrl+x`, `shift+enter`, `alt+left`,
`ctrl+shift+f`, `super+k`; letters, digits, arrows, editing keys, function keys, symbols.

## Action IDs

### Editor (`tui.editor.*`)

`cursorUp` `cursorDown` `historyPrevious` `historyNext` `cursorLeft` `cursorRight`
`cursorWordLeft` `cursorWordRight` `cursorLineStart` `cursorLineEnd` `jumpForward`
`jumpBackward` `pageUp` `pageDown` `deleteCharBackward` `deleteCharForward`
`deleteWordBackward` `deleteWordForward` `deleteToLineStart` `deleteToLineEnd`
`yank` `yankPop` `undo`

### Input & selection

`tui.input.newLine` `tui.input.submit` `tui.input.tab` `tui.input.copy`
`tui.select.up` `tui.select.down` `tui.select.pageUp` `tui.select.pageDown`
`tui.select.confirm` `tui.select.cancel`

### Fullscreen transcript (`tui.altScreen.*`)

`pageUp` `pageDown` `halfPageUp` `halfPageDown` `lineUp` `lineDown`
`previousPrompt` `nextPrompt` `search` `searchNext` `searchPrevious` `searchClose`
`top` `bottom`

### Application (`app.*`)

`app.interrupt` `app.clear` `app.exit` `app.suspend` `app.editor.external`
`app.clipboard.pasteImage`

### Sessions (`app.session.*`)

`new` `tree` `fork` `resume` `togglePath` `toggleSort` `toggleNamedFilter`
`rename` `delete` `deleteNoninvasive`

### Models & thinking

`app.model.select` `app.model.cycleForward` `app.model.cycleBackward`
`app.models.save` `app.models.enableAll` `app.models.clearAll` `app.models.toggleProvider`
`app.models.reorderUp` `app.models.reorderDown` (scoped model selector)
`app.thinking.cycle` `app.thinking.save` `app.thinking.toggle`

### Tools & messages

`app.tools.expand`
`app.message.copy` `app.message.followUp` `app.message.dequeue`

### Tree navigation (`app.tree.*`)

`foldOrUp` `unfoldOrDown` `editLabel` `toggleLabelTimestamp`
`filter.default` `filter.noTools` `filter.userOnly` `filter.labeledOnly` `filter.all`
`filter.cycleForward` `filter.cycleBackward`

## Extension keybindings

Extensions register shortcuts with `pi.registerShortcut()` using the same namespaced ID
convention (`keyHint()` helper + injected keybinding manager). Example: pi-mcp-adapter's
`mcp.panel.save` remaps the panel save key.
