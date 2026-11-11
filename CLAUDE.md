# Keystroke Board

Hammerspoon setup that records every keypress and every copied text item system-wide, saves them to SQLite, and shows a live dashboard (a Keychron K3 v1 keyboard that lights up, plus tables) in a borderless window on the portrait Dell monitor.

## Files

This project lives in `~/.hammerspoon/keystroke_board/`. Hammerspoon has a single config folder, so other projects sit beside this one.

- `~/.hammerspoon/init.lua` (shared entry point, outside this folder): loads `hs.ipc` (so `hs -c` works), enables autolaunch, and requires this project into the **global** `keystrokeBoard`. Keep it global: a local event tap or timer gets garbage collected and silently stops. Other projects need their own global names and must not reuse this project's hotkey.
- `init.lua` (this folder): the module. Key capture, clipboard polling, SQLite, the webview window, the menu bar item and the hotkey. `require("keystroke_board")` finds it through Hammerspoon's `?/init.lua` search path.
- `board.html`: the dashboard. A plain local page loaded with `file://` in an `hs.webview`, found through `MODULE_DIR`, so the folder can be renamed. It has no storage of its own; Lua owns all state.

Data: `keys.db` in this folder, next to `board.html` (WAL mode, so `keys.db-wal` and `keys.db-shm` appear beside it). `.gitignore` here excludes all of them: the file holds every keypress and copied text, so it must never be committed. Keep those ignore rules if you rename the file or add another database.

## Architecture

Lua is the source of truth. The page only renders what Lua sends and asks Lua to do things.

- **Lua → page:** `wv:evaluateJavaScript("KB.batch([...])")`. Messages are queued with `ui(msg)` and sent together on a `hs.timer.doAfter(0)` tick, so the event tap callback never waits on the webview. Use `sendNow` only outside the tap (snapshot, clear).
- **Page → Lua:** `window.webkit.messageHandlers.kb.postMessage({ action, ... })`, handled in the `hs.webview.usercontent` callback in `createWindow`.
- **Paged replies go through `ui()`**, not `sendNow`, so they arrive after any presses already queued. The page's live-update offset logic depends on that order.

### Messages

| Lua → page `type` | Meaning |
|---|---|
| `snapshot` | Full reload: counts, first history page, first clips page, totals, `dbPath`, `historyDays` |
| `historyPage` / `clipsPage` | `{ offset, total, rows }` for a requested page |
| `record` | One new keypress `{ t, label }` |
| `clip` | One new clipboard item `{ id, t, kind, app, preview, length }` |
| `copied` | A saved clip was put back on the clipboard `{ id }` |
| `down` / `up` / `flash` | Key highlight on the board, by web `KeyboardEvent.code` |
| `status` | `listening` · `paused` · `secure` (password field) · `noaccess` (no Accessibility permission). The page starts as `connecting`. |

| Page → Lua `action` | Effect |
|---|---|
| `togglePause` | Pause or resume both key and clipboard recording |
| `clear` | Delete all presses, counts and clips |
| `history` / `clips` | Request a page `{ offset, limit }` |
| `copyClip` | Put a saved clip back on the clipboard `{ id }` |
| `deleteClip` | Delete one clip `{ id, offset, limit }`, reply with the current page (steps back one page if it became empty) |
| `clearClips` | Delete all clips only |

## Database

```sql
presses (ts INTEGER, label TEXT)              -- every press, ms since epoch; pruned after HISTORY_DAYS (7)
counts  (label TEXT PRIMARY KEY, n INTEGER)   -- all-time totals, never pruned
clips   (id INTEGER PRIMARY KEY, ts INTEGER, kind TEXT, content TEXT, app TEXT)  -- pruned after 7 days
```

- **Press writes are buffered** and flushed in one transaction every 2s (`flush()`). Call `flush()` before any read that must include recent presses.
- **Clips are inserted right away**, since copies are rare.
- **Prepared statements** (`insertPress`, `bumpCount`, `insertClip`) are reused: `bind_values` → `step` → `reset`.

## Key capture concepts

- **Event tap types:** `keyDown`, `keyUp`, `flagsChanged` (modifiers) and `systemDefined` (media keys). The callback always returns `false` so it never swallows a key, and it is wrapped in `pcall`.
- **Keycodes:** `CODES` maps macOS virtual keycodes to web `KeyboardEvent.code` names, because the board layout in `board.html` is keyed by web codes. `NAMES` gives the label used in combos.
- **Media keys:** in Mac mode the K3's F-row sends media keys; `MEDIA` maps them back to `F1`–`F12`. Keycodes 160 and 131 (Mission Control, Launchpad) map to F3 and F4.
- **fn:** the physical key is keycode **63**, a `flagsChanged` event. The `fn` flag turning on means pressed (lit and recorded), turning off means released. It's recorded as its own key `Fn`, not as a combo modifier. The automatic `fn` flag macOS adds to arrow and F-key presses is ignored in combo labels.
  - Keycode **179** (`FN_TAPPED`) is *not* the key. macOS sends it as a down+up pair at the moment a short fn tap is released (the Globe action), and never during a hold. It's ignored, because handling it would count each tap twice and only flash on release. This was confirmed by logging raw events from the K3.
- **Minimum highlight:** the board keeps any key lit for at least 150ms (`MIN_LIT`), so a down and up that arrive together are still visible.
- **Labels:** combos are recorded as one label joined with `+`, modifiers in macOS order `⌃ ⌥ ⇧ ⌘` (e.g. `⌘+C`, `⇧+;`). The page splits labels on `+` to draw keycaps, so a key name must never contain `+`.
- **Modifier pressed alone** (`soloMod`): recorded on release as e.g. `⇧+Shift`, but only if no other key joined it.
- **Left vs right modifiers:** `heldMods` tracks each side. When a class's flag disappears, both sides are released.
- **Caps Lock:** one `flagsChanged` per physical press, so it is recorded on each event and only flashed on the board.
- **Not recorded:**
  - autorepeat (highlight only);
  - the K3 light key, which macOS never sees;
  - anything typed while Secure Input is on (password fields).
- **Watchdog** (every 2s): restarts the tap if macOS disabled it, and reports status. It skips `tap:start()` when `hs.accessibilityState()` is false, so the console isn't flooded with errors.

## Clipboard concepts

- **Polling:** `hs.pasteboard.changeCount()` is checked every `CLIP_POLL` (0.5s). The count at startup is taken as the baseline, so whatever was on the clipboard before startup isn't recorded.
- **Text only.** A copy that includes `public.file-url` is skipped: Finder file copies also carry the file name as text. Invalid UTF-8 is skipped because `hs.json` can't encode it.
- **Skipped as secrets:** `SKIP_PASTEBOARD_TYPES` holds nspasteboard.org markers (Concealed, Transient, AutoGenerated) and 1Password's type. Never remove these.
- **Repeats:** `lastClip` skips the same text copied twice in a row. `copyClip` sets it before writing to the clipboard so a restored item isn't saved again. Delete and clear reset it to `nil`.
- **Limits:** text over `CLIP_MAX_CHARS` is truncated with `utf8Cut`, never a byte `sub`. Only `CLIP_PREVIEW` characters are sent to the page.
- **Paused** means nothing is recorded, but `lastChangeCount` still advances.

## Board (board.html) conventions

- **Layout:** sized for a 1080×1920 portrait screen. The page scrolls if content is taller than the screen.
- **Keyboard:** built from `ROWS` as `[code, legend, width-in-units, opts]` on a 16-unit grid. Sizes use container query units (`cqw`) so it scales with width. The keyboard case stays dark in both themes.
- **Colors:** every color is a CSS custom property on `:root`, redefined under `prefers-color-scheme: dark`.
- **Paged lists:** `makeFeed()` serves both History and Clipboard (both tables; Clipboard columns are Time, Origin = source app, Content, actions). Offset 0 is the live page: new rows are prepended there. On older pages `offset++` keeps the view still while new items arrive. Every feed passes an `empty()` element factory.
- **Most used:** paged on the page itself from the full `counts` map.
- **Fixed table height:** `.rows` reserves the header plus exactly 10 rows (`--head-h: 36px`, `--row-h: 40px`), so panels never change height. Cells don't wrap. If you change rows per page, change `PAGE` in JS, `HISTORY_PAGE` in Lua and the `10 *` in `.rows` together.
- **Pager text:** must stay on one line. The buttons are compact (`‹` `›` and **Latest**), and `.info` truncates with an ellipsis.
- **Destructive buttons:** use `confirmButton(btn, action)`, which needs a second click within 3s. The webview can't show `confirm()`.
- **Copy:** user-facing text says what happens ("Clear clipboard history", "Copied"). Write units out ("presses", "chars") and handle the singular.

## Working on it

- **Reload Lua:** `hs -c 'hs.reload()'`, then check the console for errors: `hs -c 'print(hs.console.getConsole():sub(-600))'`.
- **Reload only the page:** `hs -c 'keystrokeBoard.wv:reload()'`. The snapshot is re-sent on `didFinishNavigation`.
- **Inspect the page from the terminal:** `evaluateJavaScript` is async, so store the result in a global and read it in a second call:
  ```sh
  hs -c 'keystrokeBoard.wv:evaluateJavaScript("document.title", function(r) _kbCheck = tostring(r) end)'
  hs -c 'print(_kbCheck)'
  ```
  You can also right-click the board and choose Inspect Element (`developerExtrasEnabled` is on).
- **Query data:** `sqlite3 ~/.hammerspoon/keystroke_board/keys.db`.
- **Testing touches real data.** Back up the clipboard (`pbpaste`) before tests and restore it afterwards. Delete test rows by their exact content, and check that the restore didn't add a duplicate clip. Never run Clear actions against the user's real data to test them.
- **Requirements:** Hammerspoon needs Accessibility permission for the event tap (System Settings → Privacy & Security → Accessibility). The clipboard works without it.

## User-facing controls

- `⌃⌥⌘K`: show or hide the board.
- **⌨ menu bar item:** Pause/Resume, Show/Hide board, Open data folder, Reload config.
- **Board header:** Clear all data (left), then the Listening/Paused toggle.
- **Clipboard header:** Clear clipboard history. Each row has Copy and Delete.
