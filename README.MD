# settings_mask

A GTK4/Libadwaita settings UI for **Hyprland v0.55+ Lua configurations**.  
All settings are controlled via a `settings.ini` file — no Python knowledge required to add new settings.

---

## Screenshots

> Sidebar navigation on the left, settings on the right, Matugen color scheme applied automatically.
![Autostart](assets/Autostart.png)
![Keybindings](assets/Keybindings.png)
![General settings](assets/General.png)
![Variables](assets/Variables.png)
![Windowrules](assets/Windowrules.png)
---

## Requirements

```bash
sudo pacman -S python-gobject libadwaita
```

Optional for automatic theming:
```bash
# Matugen must have generated colors.css at:
~/.themes/Matugen/gtk-4.0/colors.css
```

---

## Installation

```bash
git clone https://github.com/YOURUSER/settings_mask
cd settings_mask
chmod +x settings_mask.py
```

Edit `settings.ini` to point to your own Hyprland config files, then run:

```bash
./settings_mask.py settings.ini
```

---

## Localization

The tool is completly localizable in any language. Just create a new localization in the i18n folder, and localize your settings.ini.

The tool should automatically use your localization. If it will not pick up your localization just start it with:

```bash
./settings_mask.py settings.ini --lang de
./settings_mask.py settings.ini --lang en
```

---

## What does it do?

`settings_mask` is a graphical interface for Hyprland Lua configuration files.  
It reads your config files directly, displays the values in a structured UI, and writes changes **in-place** back to disk — without touching anything else in the file.

Supported features:

- Simple numeric values (slider)
- Selection lists (dropdown)
- Multi-field entries with any number of fields (keybinds, window rules, etc.)
- Global variables (`KEY = VALUE`)
- Autostart entries
- Animations, workspace rules, layer rules
- Creating new entries and deleting existing ones

The color scheme is loaded automatically from Matugen. Dark mode is always active.

---

## Structure of `settings.ini`

Each section in the INI corresponds to one setting or group of settings.  
All paths must be absolute.

### Common fields

| Field | Description |
|-------|-------------|
| `category` | Sidebar category (same name = same page) |
| `label` | Display name in the UI |
| `description` | Optional subtitle |
| `type` | Field type (see below) |
| `config_file` | Path to the Lua file to read/write |
| `config_key` | Key to search for in the file |
| `format` | Write format: `lua` (number) or `lua_string` (quoted string) |
| `visible` | `false` hides the section from the UI (used for templates) |

---

## Field types

### `type = slider`

Displays a slider. Clicking "Apply" writes the value to the file.

```ini
[gaps_out]
category     = General
label        = Outer Gaps
description  = Distance between windows and screen edge
type         = slider
min          = 0
max          = 40
step         = 1
config_file  = /home/user/.config/hypr/general.lua
config_key   = gaps_out
format       = lua
```

Lua file before:
```lua
gaps_out = 10,
```
After changing to 20:
```lua
gaps_out = 20,
```

**Additional fields:**

| Field | Description |
|-------|-------------|
| `min` | Minimum value |
| `max` | Maximum value |
| `step` | Step size |
| `parent_key` | Only search for the key inside this Lua block (e.g. `input`) |

---

### `type = dropdown`

Displays a selection menu with predefined values.

```ini
[accel_profile]
category     = Input
label        = Acceleration Profile
type         = dropdown
options      = flat, adaptive, custom
config_file  = /home/user/.config/hypr/general.lua
config_key   = accel_profile
parent_key   = input
format       = lua_string
```

**Additional fields:**

| Field | Description |
|-------|-------------|
| `options` | Comma-separated list of choices |
| `parent_key` | Only search inside this Lua block |

---

### `type = multisetting`

Displays an expandable entry with multiple fields. Used for keybinds, window rules, animations, workspace rules and layer rules.  
Each entry in the file gets its own row, identified by `index_field`.

```ini
[keybinds_view]
category     = Keybindings
label        = Edit Keybind
type         = multisetting
config_file  = /home/user/.config/hypr/keybindings.lua
config_key   = description
index_field  = description
index_value  =
format       = lua_string
fields       = [
    {"key": "description", "label": "Description",     "type": "text"},
    {"key": "use_mainmod", "label": "Use mainMod",      "type": "dropdown", "options": ["yes", "no"]},
    {"key": "modifier",    "label": "Extra Modifier",   "type": "dropdown", "options": ["none", "CTRL", "ALT", "SHIFT", "SUPER"]},
    {"key": "key",         "label": "Key",              "type": "text"},
    {"key": "command",     "label": "Command",          "type": "dropdown", "options": ["hl.dsp.exec_cmd", "hl.dsp.window.close"]},
    {"key": "argument",    "label": "Argument",         "type": "text"}
  ]
```

**Additional fields:**

| Field | Description |
|-------|-------------|
| `index_field` | Key that uniquely identifies each block (e.g. `description`, `name`, `leaf`) |
| `index_value` | Specific value to target (e.g. `DP-1`). Empty = auto-read all occurrences |
| `lua_func` | Lua function for the builder (e.g. `hl.animation`, `hl.workspace_rule`) |
| `fields` | JSON array of editable fields (see below) |
| `new_template` | Template for new entries: `__keybind__`, `__windowrule__`, `__lua_func__` |
| `append_file` | File that new entries are appended to |

#### `fields` JSON format

Each field in the array supports:

| Key | Description |
|-----|-------------|
| `key` | Lua key in the file |
| `label` | Display name in the UI |
| `type` | `text` or `dropdown` |
| `options` | Only for `dropdown`: list of choices |
| `block` | Groups the field into a Lua sub-block (e.g. `"block": "match"` → `match = { class = "..." }`) |

#### `block` fields

Fields sharing the same `block` name are grouped in the UI (indented ExpanderRow with the block name as heading) and combined into a sub-object in the Lua output:

```ini
fields = [
    {"key": "name",  "label": "Name",          "type": "text"},
    {"key": "class", "label": "Class (Regex)",  "type": "text", "block": "match"},
    {"key": "title", "label": "Title (Regex)",  "type": "text", "block": "match"},
    {"key": "blur",  "label": "Blur",           "type": "dropdown", "options": ["", "true", "false"]}
  ]
```

Produces:
```lua
hl.layer_rule({ name = "Test", match = { class = "^rofi$", title = "launcher" }, blur = true })
```

---

### `type = variable`

Reads all `KEY = VALUE` lines from the file. Each variable gets its own row with an input field. Values are written exactly as entered — `"kitty"` with quotes stays `"kitty"`, `kitty` without stays `kitty`.

```ini
[variables]
category     = Variables
label        = Variables
type         = variable
config_file  = /home/user/.config/hypr/variables.lua
```

New variables can be added via `+`, existing ones deleted via the trash icon.

---

### `type = autostart`

Reads all `hl.exec_cmd(...)` lines inside `hl.on("hyprland.start", ...)`. New entries are inserted before `end)`.

```ini
[autostart]
category     = Autostart
label        = Autostart Programs
type         = autostart
config_file  = /home/user/.config/hypr/autostart.lua
```

---

## Creating new entries

For `multisetting` types, a `+` button opens a dialog. For this to work, a second section with `visible = false` and `new_template` must exist:

```ini
[keybinds_new]
visible      = false
category     = Keybindings
type         = multisetting
config_file  = /home/user/.config/hypr/keybindings.lua
append_file  = /home/user/.config/hypr/keybindings.lua
new_template = __keybind__
fields       = [ ... ]  # same fields as the _view section
```

#### Available templates

| `new_template` | Used for |
|----------------|----------|
| `__keybind__` | `hl.bind(...)` keybinds |
| `__windowrule__` | `hl.window_rule({...})` window rules |
| `__lua_func__` | Any single-line `hl.xxx({...})` call (animations, workspace rules, layer rules, etc.) |

For `__lua_func__`, `lua_func` must also be set:

```ini
new_template = __lua_func__
lua_func     = hl.workspace_rule
```

---

## Searching inside nested Lua structures

When a key is not at the start of a line but inside a Lua block, two mechanisms are available:

**`parent_key`** — searches for the key only inside the specified block:
```ini
config_key   = sensitivity
parent_key   = input
```
Finds `sensitivity` only inside `input = { ... }`.

**`index_field` + `index_value`** — identifies the block by a unique value:
```ini
index_field  = output
index_value  = DP-1
```
Finds the block containing `output = "DP-1"` and reads/writes there.

---

## Matugen integration

The tool automatically loads `colors.css` from the following paths (first found is used):

```
~/.themes/Matugen/gtk-4.0/colors.css
~/.cache/matugen/colors.css
~/.config/matugen/colors.css
```

The accent color, slider highlight, Apply button and the active sidebar category are all colored with the Matugen primary color. Without `colors.css` the tool falls back to Adwaita Dark.

---

## File safety

All write operations are performed **in-place** (`open("r+")` + `seek(0)` + `truncate()`). The file's inode is preserved so Hyprland detects changes via inotify and reloads the configuration immediately.

---

## Extensibility

New setting types can be added **through the INI alone**, as long as the Lua syntax is single-line and flat (`hl.xxx({ key = val, ... })`).  
Nested sub-blocks are supported via `"block": "blockname"` in the `fields` definition.

Special formats like `hl.bind()` (with `..` concatenation) or multi-line blocks require extending the Python script.

---

## License

MIT
