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
- **Keep a reference to every `hs.timer` and `hs.task`** until it has fired or finished (`uiTimer`, `reviewTasks`, `limitsTask`). An unreferenced one can be garbage collected before it runs. For `ui()` that left `uiScheduled` stuck at `true`, and every later message to the page was silently dropped.
- **Errors on the page:** `KB.batch` handles each message in its own `try`, so one failing message can't drop the rest of its batch. `openHud` catches errors while building a popup and still opens it, with the error at the bottom. Both store the last error in `window.kbLastError`, which is the way to read it from the terminal: WebKit reports errors on `file://` pages only as "Script error.".
- **Page → Lua:** `window.webkit.messageHandlers.kb.postMessage({ action, ... })`, handled in the `hs.webview.usercontent` callback in `createWindow`.
- **Paged replies go through `ui()`**, not `sendNow`, so they arrive after any presses already queued. The page's live-update offset logic depends on that order.

### Messages

| Lua → page `type` | Meaning |
|---|---|
| `snapshot` | Full reload: counts, first history page, first clips page, totals, `dbPath`, `historyDays`, `claude` (all Claude Code records), `claudePrompts` (all your prompts), `claudeDays`, `claudeLimits` (last plan-limits message, if any), `claudeReviews` (prompt reviews so far), `tab`, `faceHeight` (mascot height in px, if you resized it), `user` (macOS account name, for the title) |
| `historyPage` / `clipsPage` | `{ offset, total, rows }` for a requested page |
| `record` | One new keypress `{ t, label }` |
| `clip` | One new clipboard item `{ id, t, kind, app, preview, length }` |
| `copied` | A saved clip was put back on the clipboard `{ id }` |
| `down` / `up` / `flash` | Key highlight on the board, by web `KeyboardEvent.code` |
| `claude` | New Claude Code replies and prompts, and prompts whose turn just ended `{ records, prompts, ends: [{ id, done }], tools: [{ id, tools }] }` (see Claude Code usage) |
| `promptCopied` | A prompt was put on the clipboard `{ id }` |
| `promptReview` | A prompt review started or finished `{ review: { id, status: running \| done \| error, text, error, cost, ms } }` |
| `textCopied` | A `copyText` finished `{ key }` |
| `callDetail` | One API call read back from its transcript `{ id, detail }` or `{ id, error }` (see Call details) |
| `claudeLimits` | Plan limits `{ limits: [{ kind, label, percent, resetsAt, severity }], shares: [{ name, percent }], extra, at, error }` |
| `status` | `listening` · `paused` · `secure` (password field) · `noaccess` (no Accessibility permission). The page starts as `connecting`. |

| Page → Lua `action` | Effect |
|---|---|
| `togglePause` | Pause or resume both key and clipboard recording |
| `clear` | Delete all presses, counts and clips |
| `history` / `clips` | Request a page `{ offset, limit }` |
| `copyClip` | Put a saved clip back on the clipboard `{ id }` |
| `copyPrompt` | Put a prompt's full text on the clipboard `{ id }`, without recording it as a clip |
| `reviewPrompt` | Run the prompt-review skill on a prompt `{ id }` (see Prompt review) |
| `copyText` | Put any text on the clipboard `{ text, key }` (a review's rewritten prompt), without recording it as a clip |
| `callDetail` | Read one API call in detail `{ id }` (message id) |
| `deleteClip` | Delete one clip `{ id, offset, limit }`, reply with the current page (steps back one page if it became empty) |
| `clearClips` | Delete all clips only |
| `tab` | Remember the tab shown `{ tab: "personal" \| "claude" }` in `hs.settings` (`TAB_SETTING`) |
| `faceHeight` | Remember the mascot's height `{ h }` (200–2000px) in `hs.settings` (`FACE_SETTING`) |

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

## Claude Code usage

- **Source:** Claude Code's session transcripts, `~/.claude/projects/**/*.jsonl`. Read-only: nothing is written to `keys.db`, and Clear never touches them.
- **Polling:** `pollClaude()` every `CLAUDE_POLL` (30s). `claudeFiles` keeps the byte offset read per file, so only new whole lines are parsed. Files not modified in `CLAUDE_DAYS` (30, Claude Code's own default retention) are skipped.
- **One record per reply:** `assistant` lines with `message.usage`. A reply is logged once per content block, and resumed sessions copy old history into the new file, so `claudeSeen` dedupes by `message.id|requestId`. `<synthetic>` models are skipped.
- **Record:** `{ t, id: message id, m: model, p: project (cwd basename), s: session, i: input, w5/w1: cache writes 5m/1h, r: cache reads, o: output, fast, tools }`.
- **Tools per call:** `tools` lists the tool names the reply asked for, in order (`claudeReplyTools()`). Tool calls are usually on later lines of the reply than the one that made the record, so `claudeSeen` keeps each reply's record (not `true`) and later lines add to it. If the record was already sent, the names go to the page by message id in the `claude` message's `tools`. `claudeToolIds` (weak keys) remembers the `tool_use` ids counted, so a resumed session's copy of a line doesn't add a tool twice.
- **Call details** (`callDetail(id)`): one call read back from its transcript on demand, nothing kept.
  - **Where to read:** while polling, `claudeLine()` gets each line's byte offset. `claudeFileReply` tracks where the latest reply in each file starts, and `claudeCallLoc[id]` keeps `{ path, from, at }`: the previous reply's start and this one's. A click reads only from the previous reply to this reply's tool results, never the whole file, so key capture never waits on it.
  - **Skipped lines:** `api-request-blob` lines (request bodies that repeat earlier content and tool ids without times) and sidechain lines.
  - **What it returns:**
    - `parts`: thinking, text (preview `CALL_TEXT_PREVIEW`) and tool calls, each with its log time, a one-line input (`toolSummary()`, `CALL_INPUT_PREVIEW`), its input size, and the tool result's size and time.
    - `added`: what came in since the previous reply, such as your prompt, tool results, a compaction summary and notes; attachment types go in `notes`.
    - From the previous reply: `prevRead`, `prevStart` and `prevEnd`.
    - From the request line: `requestAt`, `effort` and `thinking`.
    - `compact`: the `compact_boundary` metadata, if a compaction happened since the previous reply.
    - From usage: thinking tokens, stop reason, service tier and speed.
  - **Log timing:** Claude Code logs each part when it finishes. Thinking followed by text is logged together with the text, so the page merges parts logged within 50ms into one timeline segment. The thinking text itself isn't saved, only its token count.
- **Your prompts:** `type:"user"` lines with `origin.kind == "human"` (`claudePrompt`). That one rule leaves out tool results, slash commands, skill text, `isMeta` caveats, compact summaries and interruption markers, which are all "user" lines too. Deduped by `uuid` (`u:` keys in `claudeSeen`). `claudePrompts` holds `{ id, t, p, s, text, len, done }` with `text` cut to `CLAUDE_PROMPT_PREVIEW` (2000 chars) for the page; the full text lives only in `claudePromptFull` (Lua memory, pruned with the 30-day window) and is used by `copyPrompt`. Nothing about prompts is written to `keys.db`.
- **How long a prompt took:** Claude Code writes a `system` line with `subtype:"turn_duration"` when a turn ends. `claudeTurnEnd()` gives its timestamp to the latest prompt read in the same file (`claudeFilePrompt[path]`) as `done`, and the page shows `done - t`. Its `durationMs` isn't used: it leaves out time spent waiting on you (permissions, questions), and "Took" means from sending to finished. Interrupted turns log no end, so their prompts never get `done`. A prompt copied into a resumed session's file is deduped, and its turn end is skipped with it.
- **Prompt review:** the popup's Review button sends `reviewPrompt`. `reviewPrompt()` runs headless Claude Code through `hs.task`: `claude -p --output-format json --no-session-persistence --tools Skill --strict-mcp-config`, from the temp folder, with an instruction to use the `prompt-review` skill plus the full prompt in `<prompt>` tags on stdin.
  - The binary comes from `CLAUDE_BIN_PATHS`.
  - Only the Skill tool, and no saved session, so it can't touch files and doesn't appear on the Claude tab. It still uses the plan (about $0.10 per review, API-equivalent).
  - One run per prompt at a time; killed after `REVIEW_TIMEOUT` (180s).
  - Results (`result`, `total_cost_usd`, `duration_ms` from the JSON) live in `claudeReviews`, in memory only, pruned with the prompts.
  - The page renders the skill's markdown with `renderMd()` (escapes first, then headings, lists, tables, code blocks, bold, italic, code) in a "Prompt review" section between "Your prompt" and "Session". Code blocks get a Copy button (`copyText`).
- **Copying back:** `restoreToClipboard()` sets `lastClip` before writing, so neither a restored clip nor a copied prompt is saved as a new clip.
- **Cost is computed on the page** from `PRICES` (API list prices per million tokens; cache writes 1.25x/2x input, fast mode 2x). It's an API-equivalent estimate: a plan isn't billed per token. Add new models to `PRICES` (more specific patterns first).
- **Plan limits** (session, weekly, per-model weekly, and this week's share by product): `fetchClaudeLimits()` every `CLAUDE_LIMITS_POLL` (5 min) calls `CLAUDE_LIMITS_URL` (the undocumented endpoint behind Claude Code's `/usage`, header `anthropic-beta: oauth-2025-04-20`).
  - **Token:** read from the Keychain item `Claude Code-credentials` (`claudeAiOauth.accessToken`) with `/usr/bin/security` through `hs.task`, on every request. Never store, print or log it, and **never refresh it**: refreshing would rotate the token out from under Claude Code. When it expires (401/403) the page asks you to open Claude Code, which refreshes it.
  - **Failures** keep the last good numbers and set `error` (`nologin` · `expired` · `unavailable`); the page shows them as stale.
  - The response has many other fields (mostly null codenames). `limitsFromResponse()` reads only `limits`, `seven_day_breakdown.rows` and `spend`; guard every field with `tableOr`, since the shape isn't documented.
- **Page:** "Claude plan limits" sits above the usage panels: one meter per limit, plus a footer with this week's split by product and extra usage (only when enabled). "Claude Code usage" has Today / 7 days / 30 days tiles and a 14-day cost chart (hover a bar for that day). "Breakdown · projects" lists projects by last activity, newest first (projects only, 5 per page, paged on the page itself); clicking a row selects it (default: the most recent one, kept by name until the page reloads).
  - **Project detail** (the selected project): tiles for est. cost, sessions, prompts, replies, output tokens and last active; a token line (input, cache writes, cache reads, output, models by share of cost); its own 14-day chart. Both charts use `drawDays()`.
  - **Your prompts:** the selected project's prompts, newest first, with Took (sent to end of turn, `promptTook()`; – while running or when interrupted), Tokens (all four types summed) and Est. cost per prompt. Usage is logged per reply, not per prompt, and replies carry no `promptId`, so `promptUsage()` gives each reply to the latest prompt at or before it in the same session (by time, so other files of the session count). `PROMPT_PAGE` (8) single-line rows per page in a fixed-height `.ptable-wrap`. Clicking a row opens `openPrompt()` in the HUD popup: Usage (took, tokens, est. cost, API calls, split by type), Your prompt (full text, selectable; over 2000 chars it says so and Copy gets the full text), Prompt review (once requested), API calls (`callTable()`: every reply `promptUsage()` gave the prompt, oldest first, with time and seconds after the prompt, model and the tools the call asked for (`toolList()`: "Read ×2, Grep"; MCP tools by their own name), the four token types, est. cost and a total row for all calls; paged by `CALL_PAGE` (10) with a `‹ ›` pager when there are more, keeping a full page's height; click a row to open the call popup) and Session, with Review and Copy in the popup header. New prompts keep an older page still (`promptOffset++`).
  - All of this is on the Claude tab.

## Clipboard concepts

- **Polling:** `hs.pasteboard.changeCount()` is checked every `CLIP_POLL` (0.5s). The count at startup is taken as the baseline, so whatever was on the clipboard before startup isn't recorded.
- **Text only.** A copy that includes `public.file-url` is skipped: Finder file copies also carry the file name as text. Invalid UTF-8 is skipped because `hs.json` can't encode it.
- **Skipped as secrets:** `SKIP_PASTEBOARD_TYPES` holds nspasteboard.org markers (Concealed, Transient, AutoGenerated) and 1Password's type. Never remove these.
- **Repeats:** `lastClip` skips the same text copied twice in a row. `copyClip` sets it before writing to the clipboard so a restored item isn't saved again. Delete and clear reset it to `nil`.
- **Limits:** text over `CLIP_MAX_CHARS` is truncated with `utf8Cut`, never a byte `sub`. Only `CLIP_PREVIEW` characters are sent to the page.
- **Paused** means nothing is recorded, but `lastChangeCount` still advances.

## Board (board.html) conventions

- **Layout:** sized for a 1080×1920 portrait screen. The page scrolls if content is taller than the screen.
- **Tabs:** the header stays on both; below it, **Personal** (pixel mascot, keyboard, totals, Most used, History, Clipboard, the `keys.db` path) and **Claude** (plan limits, Claude Code usage, Breakdown). Each tab is a mission-control card (index, name in `--font-hud` Oxanium, live readout: presses and keys for Personal, session and weekly % for Claude, or today's cost when limits are unavailable); the active one has HUD corner brackets, a `--hud-grid` background and a lit bottom edge. `showTab()` switches them and the subtitle. The Listening/Paused toggle and **Clear all data** live in a toolbar at the top of Personal, since both only concern personal recording. The snapshot's `tab` restores the last one after a reload. Live updates keep flowing to the hidden tab.
- **Pixel mascot:** an original blue 28×21 manga-style dragon head (horizontal blue horns, side fins, big glossy amber eyes, ω mouth) (`FACE`, one character per pixel, colors in `FACE_COLORS`) drawn by `drawFace()` as SVG rects with `crispEdges`, in `.face` above the keyboard. Fixed colors, like the keyboard case. It is at least 200px tall (`min-height`) and resizable by dragging the corner (`resize: vertical`); a `ResizeObserver` sends the new height (`faceHeight`) and the snapshot restores it. It's an original character: don't swap in a copyrighted one.
- **Keyboard:** built from `ROWS` as `[code, legend, width-in-units, opts]` on a 16-unit grid. Sizes use container query units (`cqw`) so it scales with width. The keyboard case stays dark in both themes.
- **Colors:** every color is a CSS custom property on `:root`, redefined under `prefers-color-scheme: dark`.
- **Mission-control style** (shared by the whole page): the title, tab names and panel titles use `--font-hud` (Oxanium) in spaced capitals, panel titles have an accent bar on the left, `.panel` and `.now` get corner brackets (`--bracket`, `::before`/`::after`, so don't use those pseudo-elements on panels for anything else), stat tiles and meters get the `--hud-grid`, the page gets the fainter `--page-grid`, buttons are monospace capitals, and corners are 4px. The keyboard keeps its own rounded look because it depicts the real K3. Text values and data stay in IBM Plex Mono.
- **Fonts:** Oxanium (`--font-hud`) for the title, tab names, panel titles and popup terms; IBM Plex Mono (`--font-mono`) for all other text, prose included. Never leave new text to inherit the body font (`--font-ui`, Schibsted Grotesk): set `--font-mono` or `--font-hud` on it or on its container. Popups get this from `.hud-panel`, which sets `--font-mono` for everything inside. After a change, check computed fonts on the page, not just the CSS.
- **Paged lists:** `makeFeed()` serves both History and Clipboard (both tables; Clipboard columns are Time, Origin = source app, Content, actions). Offset 0 is the live page: new rows are prepended there. On older pages `offset++` keeps the view still while new items arrive. Every feed passes an `empty()` element factory.
- **Most used:** paged on the page itself from the full `counts` map.
- **Fixed table heights:** every table reserves room for a full page, so panels never change height. `--head-h` (36px) is shared by all headers.
  - **Most used and History:** 4 rows of `--row-h` (40px) in `.rows`. Cells don't wrap. To change the page size, change `PAGE` in JS, `HISTORY_PAGE` in Lua and the `4 *` in `.rows` together.
  - **Clipboard:** 5 single-line rows of `--row-h` (40px) in `.cliptable-wrap`. Each preview is one line: newlines show as spaces, long text ends in an ellipsis, and the full preview is in the hover tooltip. Add 1px because the header's bottom border collapses into the first row. To change the page size, change `CLIP_PAGE` in JS, `CLIP_PAGE` in Lua and the `5 *` in `.cliptable-wrap` together. `makeFeed` takes a `size` option for this.
- **Pager text:** must stay on one line. The only buttons are `‹` and `›`, and `.info` truncates with an ellipsis.
- **Destructive buttons:** use `confirmButton(btn, action)`, which needs a second click within 3s. The webview can't show `confirm()`.
- **API call popup** (`#hudCall`, `openCall()` / `renderCall()`): the second HUD layer, on top of the prompt popup (slightly narrower, 56vh).
  - **Sections:** Overview (stop reason in words, naming the tools for `tool_use` (`stopText()`); duration, output, cache writes and reads, token line), Why it looks like this (`callWhy()`), Timeline (`callSegments()`), What it did, New in this call, and Request.
  - **Why it looks like this:** plain rules over the numbers: compaction, expired or rebuilt cache, first call of the prompt, big tool result, mostly thinking, long tool input or reply, slow call, slow tool, final reply.
  - **Stepping:** `‹ ›` in the header and the arrow keys step through the prompt's calls. Requests wait 150ms after the last step (`callAsk`), and stepping skips the opening animation (`.stepped`).
  - **Caching:** details are cached on the page in `callDetails`.
- **HUD popup** (`#hud`, `openHud({ layer, title, readout, actions, fill })`, `closeHud(layer)`): two layers, `hud` and `hudCall`, built from the same markup and found by class through `hudPart()`. Closing `hud` closes `hudCall` too, and Esc closes the top one. The `hud` layer is one shared popup, 60vh tall with its body scrolling, used by the glossary and prompt details. Build content with `hudItem(i, term, note)` so every entry uses the same term pattern and staggers in. Opening animation: backdrop fade, a scan line across the middle (`hudScan`), the panel opening out from that line (`hudOpen`, clip-path), the corner brackets flying out to the corners (`hudBracket`), then the entries one by one (`hudItem`, `--i` delay); closing collapses it (`hudClose`). All of it is off under `prefers-reduced-motion`. It closes on Close, a backdrop click or Esc, and when you leave the Claude tab.
- **Glossary:** the **?** button in the header (shown only on the Claude tab) opens the HUD popup with `glossary()`: a flat list of Claude-tab terms, `[term, where it appears, explanation]`. Terms use the panel-title pattern (display font, spaced capitals, accent bar). Prices in it come from `PRICES`. **When you add a metric or a term to the Claude tab, add it to `glossary()` too.**
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
- **Test the code paths you changed in the running board, not just the syntax.** `node --check` on the page script catches syntax errors only. A wrong name (a ReferenceError) shows up only when that code runs, e.g. when a popup renders with real data. Open every view you touched, then check `window.kbLastError`.
- **`hs -c` and timeouts:** read values with `return x`, not `print(x)`. An `hs -c` that is killed mid-call (e.g. by a `perl -e 'alarm N'` wrapper) can leave its `print` replacement in place, and once those stack up `print` fails with "stack overflow". A killed call may also have run already, so don't retry calls that change state.
- **Testing touches real data.** Back up the clipboard (`pbpaste`) before tests and restore it afterwards. Delete test rows by their exact content, and check that the restore didn't add a duplicate clip. Never run Clear actions against the user's real data to test them.
- **Requirements:** Hammerspoon needs Accessibility permission for the event tap (System Settings → Privacy & Security → Accessibility). The clipboard works without it.

## User-facing controls

- `⌃⌥⌘K`: show or hide the board.
- **⌨ menu bar item:** Pause/Resume, Show/Hide board, Open data folder, Reload config.
- **? (header, right, Claude tab only):** glossary of the Claude tab's terms.
- **Board header:** the title, "<macOS account name> Board" (uppercase via CSS), and a subtitle that follows the tab.
- **Pixel mascot (Personal tab):** drag its bottom-right corner to resize it; the height is kept.
- **Personal tab toolbar:** the Listening/Paused toggle (left) and Clear all data (right).
- **Clipboard header:** Clear clipboard history. Each row has Copy and Delete.
- **Tabs:** Personal / Claude, under the header.
- **Breakdown:** click a project to open its detail and prompts below.
- **Your prompts:** click a row to open the prompt's details in a popup; Copy (in the row or the popup) puts the full prompt on the clipboard (not saved to Clipboard history). Review (popup) runs the prompt-review skill and shows the result below the prompt; Copy on the rewrite copies it. In API calls, click a row to open that call in a second popup; `‹ ›` or the arrow keys step through the calls, Esc closes it.
