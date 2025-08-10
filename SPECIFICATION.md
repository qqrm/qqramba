# qqramba — v1 Product Specification

## Product name

qqramba

## Goals

* Windows 10 and 11 x64
* Core in Rust, GUI in Tauri
* No network by default, zero telemetry
* Primary feature: Double Shift converts layout of selection or last typed segment
* Optional feature: Single Shift toggles ru en layout for the foreground thread, disabled by default
* No auto layout switching in v1

## Supported layouts

* Pair en US and ru RU
* Default HKL values 00000409 and 00000419, configurable

## Functional requirements

1. Double Shift

* If there is a non empty selection: replace the selection with converted text ru↔en
* If there is no selection: replace the last typed segment from the input buffer
* Segment boundaries: walk backward from caret to first whitespace or punctuation, hard cap segment_max
* Preserve case for each letter, digits and punctuation unchanged
* If the segment contains neither Cyrillic nor Latin letters do nothing

2. Single Shift optional

* Toggle active thread layout between primary and secondary HKL

3. Exceptions

* Ignore password fields and blacklisted processes
* Clear the buffer on Enter, Esc, Tab, Alt+Tab, foreground window change, or silence timeout

4. State control

* Tray checkbox Enabled, pause and resume only via tray and GUI
* No hotkey to enable or disable the app

5. Constraints

* Without admin rights the hook does not see elevated apps
* GUI provides Restart as administrator button

## Non functional requirements

* Privacy no network dependencies, no keystrokes on disk, text buffer only in memory, optional firewall rule to block outbound
* Performance average hook callback budget ≤ 50 microseconds, RAM ≤ 50 MB
* Reliability robust with IME and dead keys, stable across window and layout changes

## Architecture and modules

Workspace layout

```
qqramba/
  Cargo.toml  workspace
  crates/
    core/  buffer, tokenization, mapping tables, converter, config validation
    win/   Windows integration: WH_KEYBOARD_LL, HKL, SendInput, clipboard, UIA, process lists
    gui/   Tauri app: tray, settings window, backend commands
```

Threads

* Hook thread WH_KEYBOARD_LL, fast updates, posts events to a channel
* Worker thread SendInput, clipboard, HKL work, never blocks the hook
* GUI thread Tauri WebView

Backend API exposed to GUI

* get_status → enabled flag, current layout, process name, buffer stats
* set_enabled(bool)
* set_config(Config)
* open_config_folder()
* restart_elevated()

## Input pipeline

1. Install global WH_KEYBOARD_LL
2. Event handling

* Printable key down appends to buffer using current HKL and modifiers
* Backspace removes last character from buffer
* Enter, Esc, Tab clear current segment
* Foreground window change clears buffer

3. Double Shift detection

* Track Shift key up events
* If two Shift ups occur within double_shift_ms and no non Shift key was pressed between them, fire Double Shift

4. Single Shift detection

* If Single is enabled and no second Shift arrives within single_shift_ms and no other key is pressed, toggle HKL

## Selection detection

* Use clipboard heuristic since there is no universal API
* Send Ctrl+C, wait debounce 10 ms, read CF_UNICODETEXT
* If text is non empty and length ≤ selection_max treat it as selection
* Save and restore original clipboard, prompt once if very large

## Conversion algorithm

1. Direction

* If text contains Cyrillic letters use ru→en
* Else if it contains Latin letters use en→ru
* Else do nothing

2. Mapping tables

* Base QWERTY↔ЙЦУКЕН for a..z and A..Z
* Examples q↔й, w↔ц, e↔у, r↔к, t↔е, y↔н, u↔г, i↔ш, o↔щ, p↔з, a↔ф, s↔ы, d↔в, f↔а, g↔п, h↔р, j↔о, k↔л, l↔д, z↔я, x↔ч, c↔с, v↔м, b↔и, n↔т, m↔ь
* Preserve case including titlecase
* Digits and punctuation copied as is, optional extended map for [] {} ; ' later

3. Injection strategies

* Method A default SendInput backspace len(segment) then type the converted string
* Method B clipboard save current clipboard, set text, Ctrl+V, restore clipboard
* Per process overrides choose A or B

4. Caret position remains at end of the replaced fragment

## Configuration

File location

* %AppData%\qqramba\config.toml
* Portable mode if config.toml sits next to the exe

Defaults

```toml
[hotkeys]
double_shift_ms = 350
single_shift_enabled = false
single_shift_ms = 360

[buffer]
max_chars = 120
segment_max = 40
silence_timeout_ms = 3000
selection_max = 4000

[layout]
primary = "00000409"
secondary = "00000419"

[behaviour]
method = "backspace"   # backspace or clipboard
clear_on_enter = true

[security]
no_network = true
clipboard_restore = true

[ignore]
process_blacklist = ["KeePass.exe","mstsc.exe","steam.exe","chrome.exe","firefox.exe","msedge.exe"]
process_whitelist = []
```

Rules

* Config changes hot reload with validation, invalid fields are logged and ignored

## GUI in Tauri

Look and feel

* Top bar accent orange similar to Rust
* Main screen with large tiles Double Shift, Single Shift with toggle, Help
* Footer shows version, About, Open source

Tray

* Icon reflects Enabled or Paused
* Menu Enabled, Pause 5 minutes, Settings, Restart as admin, Exit

Settings window

* General enable Single Shift, sliders for double_shift_ms and single_shift_ms, global Enabled
* Buffer segment_max, silence_timeout_ms, clear_on_enter
* Layout choose primary and secondary HKL, Switch now button
* Methods choose backspace or clipboard, per process overrides table
* Privacy no network explanation, button Create firewall outbound block rule
* Ignore blacklist and whitelist editors, Add from running processes button
* About version, build hash, license

Tauri hardening

* No shell open, no http
* Allow only fs inside app config folder and clipboard
* CSP default src 'self'

## Installation and packaging

* Build cargo build --release and tauri build
* Distribute portable zip and MSI
* Installer may add HKCU Run autostart and an outbound firewall block rule
* Code signing recommended

## Logging

* Local only, never include user text
* Example entries layout switched to HKL, double shift applied len N method backspace process name, clipboard restore failed

## Testing

* Unit tests mapping and case rules, property tests ru→en→ru identity for letters
* Integration tests detector for Double Shift, buffer clearing, segmentation, injection methods, process lists
* Manual matrix Notepad, VS Code, JetBrains IDE, Word, Slack, Telegram, Chrome, Firefox, Edge, IME presence, long lines, elevated apps

## Threat model

* Keystroke interception is required by design
* Mitigations open source, no network, local logs without content, minimal dependencies
* Emphasis on ignoring password fields and private apps, UIAccess build or run as admin for elevated targets

## Windows APIs

* SetWindowsHookExW WH_KEYBOARD_LL, CallNextHookEx, UnhookWindowsHookEx
* GetForegroundWindow, GetWindowThreadProcessId
* GetKeyboardLayout, LoadKeyboardLayoutW, ActivateKeyboardLayout
* SendInput, do not use keybd_event
* Clipboard OpenClipboard, GetClipboardData, SetClipboardData, EmptyClipboard, CloseClipboard
* UI Automation IsPasswordPattern, ES_PASSWORD for classic Edit

## Roadmap

* v1 Double Shift, optional Single Shift, GUI and tray, process lists, network blocked, MSI
* v1.1 per app method overrides, portable mode, improved selection detection, extended punctuation map
* v1.2 UIAccess build, IME optimizations

## Acceptance criteria v1

* Double Shift converts selection and last segment, preserves case
* Single Shift when enabled toggles HKL for the active window
* Events do not run in blacklisted processes, password fields ignored
* No outbound traffic, firewall helper works
* Double Shift reaction time ≤ 100 ms in typical apps
* No crashes when switching windows and layouts

## Appendix how auto layout switching is usually built

* Dictionary check last word against two dictionaries, convert on match
* N gram or tiny classifier estimate language of recent buffer and convert at word boundary
* Hybrid dictionary plus n grams with user feedback to adjust thresholds
* Risks false positives in code and logins, IME interference, performance cost, so it is off in v1

