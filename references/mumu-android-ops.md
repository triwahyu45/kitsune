# MuMu Player & Instagram Automation Skill

This skill captures verified architectures, commands, input pipelines, and safety guardrails for driving Android Instagram and high-speed typing automation on **MuMu Player 12** via `uiautomator2` and `adb`.

---

## 1. Scope & Trigger Context
Use this skill whenever:
- Interacting with **Instagram Android app** on an Android emulator (MuMu Player 12).
- Performing high-speed text input, human typing emulation (80–113 WPM), or benchmarking on web/native inputs (e.g., Monkeytype).
- Managing Instagram Direct Message (DM) inboxes, replying to follower curhat/chats, and posting or inspecting Instagram Stories.
- Recovering, booting, or managing MuMu Player VM instances via CLI without manual GUI intervention.

---

## 2. Verified Paths, Commands & Schemas

### A. MuMu Player 12 CLI Management
- **CLI Executable:** `C:\Program Files\Netease\MuMuPlayer\nx_main\MuMuManager.exe`
- **Default Port & Serial:** `127.0.0.1:16384`
- **Inspect VM Status:**
  ```powershell
  & 'C:\Program Files\Netease\MuMuPlayer\nx_main\MuMuManager.exe' info -v 0
  ```
  Returns JSON: `is_android_started` (bool), `adb_port` (16384), `player_state` ("start_finished").
- **Cold Boot / Launch Headless or GUI:**
  ```powershell
  & 'C:\Program Files\Netease\MuMuPlayer\nx_main\MuMuManager.exe' control -v 0 launch
  & 'C:\Program Files\Netease\MuMuPlayer\nx_main\MuMuManager.exe' control -v 0 hide_window
  adb connect 127.0.0.1:16384
  ```
- **Headless / Hidden Window Mode (No Double Window with PySide6 GUI):**
  `MuMuManager.exe control -v 0 hide_window` hides the desktop emulator window while `adb` and `uiautomator2` (`d.screenshot()`) continue capturing live `(720, 1600)` frames cleanly. Restore anytime with `show_window`.

### B. Python `uiautomator2` Connection
```python
import uiautomator2 as u2
d = u2.connect('127.0.0.1:16384')
d.app_start('com.instagram.android')
```

---

## 3. High-Speed Human Typing Engine (80–113 WPM)

### Why NOT `d.send_keys()`?
Standard `d.send_keys()` in `uiautomator2` triggers a synchronous HTTP RPC round-trip (~170–200ms per character), causing lag, blocking, and an apparent speed of only ~15–20 WPM.

### The Verified Solution: Persistent ADB Shell Pipe
Keep a single persistent `adb shell` process open and pipe keystrokes directly into its `stdin`:
```python
import subprocess
import time
import random

p = subprocess.Popen(
    ['adb', '-s', '127.0.0.1:16384', 'shell'],
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True
)

# Natural human typing jitter (80-113 WPM equivalent)
# Jitter: 0.5ms - 10ms (or 5ms - 15ms)
for char in text_to_type:
    if char == ' ':
        p.stdin.write('input text %s\n')
    elif char.isalnum():
        p.stdin.write(f'input text {char}\n')
    elif char in ['!', '?', '.', ',', ':', '-', '(', ')']:
        p.stdin.write(f'input text \"{char}\"\n')
    p.stdin.flush()
    time.sleep(random.uniform(0.005, 0.015))

p.stdin.close()
p.wait()
```

### Full-Message Fallback for Emojis
If emojis or complex unicode characters are included:
```python
# Set complete text with emojis directly on active EditText
d(className='android.widget.EditText').set_text(full_message)
d(description='Send').click()
```

---

## 4. Instagram DM Inbox Navigation & Safe Handling

### Navigation Hierarchy (`720x1600` Portrait)
- **Bottom Navigation Bar (`com.instagram.android:id/tab_bar`, `y ≈ 1491`):**
  - `Home`: `com.instagram.android:id/feed_tab` `(72, 1491)`
  - `Reels`: `com.instagram.android:id/clips_tab` `(216, 1491)`
  - `Direct Messages`: `com.instagram.android:id/direct_tab` `(360, 1491)` — ALWAYS target `resourceId="com.instagram.android:id/direct_tab"` instead of bare `desc="Message"`, because tapping `desc="Message"` inside a user's profile modal opens that user's chat in a loop instead of the DM Inbox!
  - `Search`: `com.instagram.android:id/search_tab` `(504, 1491)`
  - `Profile`: `com.instagram.android:id/profile_tab` `(648, 1491)`
- **Inbox Thread Detection (Unread + Recent Read, Tabbed Inbox, & Anti-Self-Reply Geometry):**
  - Modern Instagram Compose Inbox keeps `Filters | Primary | Requests | General` tabs active in-place (`Hidden requests` / `No message requests` remains visible after checking `Requests`). Always detect and tap `Primary`/`Utama` at scan start if stuck on `Requests`/`General`, AND switch back to `Primary` when Stage 2 finishes.
  - Inbox thread rows in Compose UI often have `resource_id=""` and `clickable=False` (wrapping clickable container has empty desc). Match both `row_inbox_container` AND full-width rows (`width >= 0.75*W`, `50 <= height <= 220`) with comma-separated `content-desc` below the `Primary` tab bar (`y >= max(0.28*H, primary_tab_y2 + 6)`, `y2 <= 0.90*H`).
  - Prioritize `", unread,"` / `"new message"` rows first, then scan top recent read rows whose preview does NOT indicate Aina already spoke last (`", you:"`, `"you sent"`, `", seen"`, `"Sent <time>"`), keeping `"Draft:"` eligible because a draft was never sent.
  - Inside chat threads, filter out `seen_state_text`, `message_action_log` (disappearing messages / vanish notices), `reply_preview`, `direct_context_reply`, `pog_bubble_text`, `Edited`, timestamps, bio headers, and composer (`Message…`), then classify bubbles geometrically (`x1 < 0.28*W and x2 < 0.85*W` = incoming friend; right-aligned = Aina). Never reply if the bottom-most bubble is already right-aligned.
  - **Adaptive Quote-Reply vs Direct Reply & SQLite Bubble Dedup:**
    - Use `dm_replied_bubbles(bubble_hash TEXT PRIMARY KEY, ...)` in `aina_infinite_memory.sqlite3` (hashing `sender|content_sig`, including `[MEDIA:<desc>]` for Reels/photos/stickers/VNs) to never reply twice to the same bubble.
    - Classify bubbles cleanly (`left < 250` = incoming, `right > 470` = outgoing Aina); if bottom-most bubble is right-aligned (`out`), **STOP immediately** and exit cleanly.
    - Auto-disable disappearing messages on thread entry via `turn_off_disappearing_messages_u2()` / `turn_off_disappearing_messages_if_active()` (`"Turn off disappearing messages"` / `"Matikan pesan sementara"`).
    - **1 unanswered bubble:** reply directly in composer (no quote swipe). **>1 unanswered bubbles:** trigger horizontal-only swipe-right (`swipe(x1 + 25, cy, min(x1 + 240, 0.72*W), cy, 0.15)` — NEVER vertical swipe-up!) + fallback `long_click -> Reply/Balas` per bubble.
    - **Live Whitelist & Spam Cleanup:** Toggle `social.dm_test_whitelist_enabled` in `/api/settings` / Web Settings UI (`["tri wahyu", "triwahyu45"]` during test runs) and use `cleanup_spam_dm.py --target "..." [--execute]` to unsend duplicate right-side Aina spam while preserving ≥ 1 valid reply.
    - **Digital Common-Sense Awareness & SQLite RL (`aina_phone_rl.sqlite3`):** `PhoneRLEngine` tracks `phone_rl_qtable` & `phone_rl_experiences` with automatic reward shaping (`+2.0` natural DM completion, `+1.0` clean nav / skipped Sponsored ad, `-2.0` inefficiency, `-5.0` accidental disappearing/ad browser, `-10.0` critical `auto_abort` on call screen / Block / Restrict / Report / Unfollow / `.apk` / Logout). Normal DM reading uses mid-screen `scroll_dm_read_older_u2()` cleanly away from bottom edge, and only allows disappearing/calls if explicitly requested in chat context.

### Safety Guardrails (CRITICAL)
- **Zero Accidental Outgoing Calls:** Never tap the top right header area (`X >= 480, Y <= 200`) inside a chat room (contains the Voice Call and Video Call icons).
- **Anti-Vanish Mode:** Never swipe up inside a thread room to prevent activating Vanish Mode.

---

## 5. Dead-Ends, Pitfalls & Verified Fixes

| Failed Attempt / Pitfall | Root Cause | Verified Fix / Hard Rule |
| :--- | :--- | :--- |
| `uiautomator2.ConnectError: device not online` | MuMu Player crashed or closed in background | Run `MuMuManager.exe control -v 0 launch`, wait for `is_android_started: true`, then run `adb connect 127.0.0.1:16384`. |
| Typing feels sluggish or blocked in chat / benchmark | `d.send_keys()` has ~170ms RPC per character | Use persistent `adb shell` pipe with `0.5ms–10ms` jitter stream. |
| Monkeytype shows low WPM / 89% AFK | Typing 1 sentence then pausing while 30s timer runs | Keep typing continuously line-by-line without stopping throughout the entire 30s window. |
| Clicking chat row opens Map / Notes modal | Tapped top bar story/notes area instead of message list | Target `", unread,"` rows with `y > 0.28 * H` and `y < 0.90 * H`. |
| DM loop stuck opening same profile's `Message` button | Tapped bare `desc="Message"` inside a user profile modal instead of DM tab | Target `com.instagram.android:id/direct_tab`, and press `back` / `app_start(..., stop=True)` if `direct_tab` is hidden. |
| Inline python quotes fail in PowerShell (`ScriptBlock should only be specified...`) | Complex multiline quotes in `python -c "..."` break in PowerShell | Write script to a `.py` file (e.g. in workspace or scratch) and run `python script.py` or `@' ... '@ | python -`. |
| DM replies contain stray `'d'`, dropped emojis, or mid-sentence sends | `aina_human_typer` used `shell=True` per-char `subprocess.run`, converted `\n` to `KEYCODE_ENTER`, and stripped non-ASCII | Use `sanitize_dm_text()`, `shell=False` single `adb shell` pipe or chunked `EditText.set_text()`, plus pre-send read-back `verify_typed_text()` -> `log_self_diagnosis()` + `phone_rl` penalty + atomic `set_text` auto-heal. |
| DM Message Requests ignored or hardcoded tap misses Requests tab | `check_and_reply_dms()` only scanned Primary unread rows and skipped threads missing composer before tapping `Accept`; `tap_xy(550, 140)` could hit wrong button | Dynamically match `Requests`/`Permintaan` bounds (excluding Hidden), accept inside thread (`Accept`/`Terima` -> `Primary`/`Utama`), re-read composer, then send verified reply. |
| Bot turns on Vanish Mode / Disappearing Messages or replies to empty Inbox UI text | `scroll_down()` / Reels `d.swipe(360,950,360,200)` swiped up inside an active DM chat, or `check_and_reply_dms()` tapped inbox headers (`Your note` / `No message requests yet`) when Requests empty | Block `scroll_down()` / swipe-up whenever `is_in_instagram_chat_thread()` is True, block `Disappearing messages`/`Vanish mode` labels in `tap_*`, abort Requests scan on empty-state markers (`No message requests`), require `row_thread_composer_edittext` before replying, and block self-replies to `ainavenara`. |
| Opened/read DM threads stay unreplied forever or Inbox gets stuck on `Requests` tab | Scanning only `", unread,"` rows skips opened-without-reply threads, Compose Inbox uses `rid='' clk=False` rows, and `Requests` stays selected in-place | Switch `Requests` -> `Primary` before/after scans, match full-width comma-desc Compose rows + `Draft:` rows, check inside-thread `bubbles[-1]` side, filter `message_action_log`, and use horizontal-only swipe-right for per-bubble quote-replies. |
| User chat discussing disappearing messages dropped as noise / Aina asks "mana yang belum dijawab" | `_is_noise_chat_text` checked bare substring `"disappearing message"`, dropping user questions about disappearing chat; quoted messages lacked parent context | Filter system logs only via `message_action_log` or `low.startswith(...)` prefixes; associate `quote_previews` with adjacent incoming bubbles (`[Membalas: "..."] <text>`); strip novel roleplay `*(...)*` via `SecurityGuard.strip_roleplay_narration()`. |
