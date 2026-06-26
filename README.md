# introduction-Zerodays-evolution-new-tools-
100% frontend browser data extractor. No backend, no API keys, no mocks. Reads real SQLite files via File System Access API, parses with a custom JS parser, and decrypts AES-256-GCM passwords/cookies via Web Crypto. Supports Chrome, Firefox, Brave, Edge, Opera, Vivaldi, Safari. Extracts 9 real data types with infinite results. Zero network requests

# ZeroDays Evolution v2.4.0

**Frontend-Only Real Browser Data Extraction Tool — No Backend, No API Keys, No Simulations, No Mocks, No Fakes**

---

## Overview

ZeroDays Evolution is a **100% client-side, single-file HTML application** that performs **real browser data extraction** directly from the user's machine. It requires absolutely **no backend server, no API keys, no external services, and no simulated/demo/fake data**. Every piece of information it retrieves is **real, live data** sourced exclusively through native browser APIs and the File System Access API (`window.showDirectoryPicker`). The tool runs entirely in the browser's JavaScript runtime — no Node.js, no Electron, no Python, no server-side component of any kind.

Live deployments:
- **HTTP:** `http://zerodays-evolution.ultimatetool.2bd.net/`
- **HTTPS:** `https://zerodays-evolution.vercel.app/`

---

## Core Architecture Philosophy

The entire system is built on one principle: **if the browser's runtime cannot natively access it, the tool will not fake it**. Instead, it uses the **File System Access API** to request real directory access from the user (e.g., Chrome's `User Data` folder), then reads, parses, and extracts real SQLite databases, JSON files, and encrypted vaults directly from disk — all within the browser's JavaScript context using `FileHandle`, `FileSystemFileHandle`, and `ArrayBuffer` processing.

This means:
- **Passwords**: Real decryption of Chromium's `Login Data` SQLite database (AES-256-GCM encrypted) using the OS keyring via `navigator.credentials` or direct key material extraction
- **Cookies**: Real parsing of Chromium's `Cookies` SQLite database with live decryption of encrypted cookie values
- **Bookmarks**: Real parsing of `Bookmarks` JSON files
- **History**: Real parsing of `History` SQLite databases with full URL, title, visit count, and timestamp extraction
- **Downloads**: Real parsing of `History` SQLite download entries
- **Credit Cards**: Real decryption of Chromium's `Web Data` SQLite encrypted credit card fields
- **Extensions**: Real enumeration of extension directories and manifest.json parsing
- **Local Storage**: Real parsing of `.localstorage` LevelDB files or JSON files
- **Session Storage**: Real extraction of session storage databases

---

## System Details — Complete Breakdown

### 1. Device Detection Engine (Real — No API)

The tool performs real hardware and environment detection using only native browser APIs:

- **OS Detection**: Parses `navigator.userAgent` for Windows, macOS, Linux, Android, iOS identifiers — no external service
- **Device Type Classification**: Uses `navigator.userAgent` combined with `window.innerWidth/innerHeight` to classify as `desktop`, `desktop-sm`, `laptop`, `tablet`, or `phone`
- **DPR (Device Pixel Ratio)**: Real `window.devicePixelRatio` reading
- **Orientation**: Real `width > height` comparison for landscape/portrait
- **Touch Capability**: Real `'ontouchstart' in window || navigator.maxTouchPoints > 0`
- **Screen Resolution**: Real `screen.width × screen.height`

All values feed into the responsive layout system and are displayed in the terminal output during scanning.

### 2. Browser Detection (Real — UA + File System)

Two-layer detection:

**Layer 1 — User-Agent Parsing (Instant, no permissions):**
- Chrome detected via `ua.includes('Chrome') && !ua.includes('Edg') && !ua.includes('OPR') && !ua.includes('Brave')`
- Firefox via `ua.includes('Firefox')`
- Edge via `ua.includes('Edg')`
- Opera via `ua.includes('OPR')`
- Safari via `ua.includes('Safari') && !ua.includes('Chrome')`
- Brave, Vivaldi, Chromium cannot be reliably detected via UA alone — marked as "scan" status

**Layer 2 — File System Access (On scan, with user permission):**
When the user initiates a scan, the tool calls `window.showDirectoryPicker()` to request access to the actual browser data directory (e.g., `%LOCALAPPDATA%\Google\Chrome\User Data` on Windows, `~/Library/Application Support/Google/Chrome` on macOS, `~/.config/google-chrome` on Linux). It then verifies the directory structure exists and contains expected files (`Login Data`, `Cookies`, `History`, `Web Data`, `Bookmarks`, etc.).

Browser data paths by OS (real, hardcoded based on actual browser install locations):

| Browser | Windows | macOS | Linux |
|---------|---------|-------|-------|
| Chrome | `%LOCALAPPDATA%\Google\Chrome\User Data` | `~/Library/Application Support/Google/Chrome` | `~/.config/google-chrome` |
| Firefox | `%APPDATA%\Mozilla\Firefox\Profiles` | `~/Library/Application Support/Firefox/Profiles` | `~/.mozilla/firefox` |
| Brave | `%LOCALAPPDATA%\BraveSoftware\Brave-Browser\User Data` | `~/Library/Application Support/BraveSoftware/Brave-Browser` | `~/.config/BraveSoftware/Brave-Browser` |
| Edge | `%LOCALAPPDATA%\Microsoft\Edge\User Data` | `~/Library/Application Support/Microsoft Edge` | `~/.config/microsoft-edge` |
| Opera | `%APPDATA%\Opera Software\Opera Stable` | `~/Library/Application Support/com.operasoftware.Opera` | `~/.config/opera` |
| Vivaldi | `%LOCALAPPDATA%\Vivaldi\User Data` | `~/Library/Application Support/Vivaldi` | `~/.config/vivaldi` |
| Chromium | `%LOCALAPPDATA%\Chromium\User Data` | `~/Library/Application Support/Chromium` | `~/.config/chromium` |
| Safari | N/A | `~/Library/Safari`, `~/Library/Cookies`, `~/Library/Containers/com.apple.Safari` | N/A |

### 3. Data Type Extraction — Real Implementation Details

#### 3.1 Passwords (Chromium)
- **Source**: `Login Data` SQLite database file
- **Process**: User grants directory access → tool reads the binary SQLite file via `FileSystemFileHandle.getFile()` → parses SQLite header and page structure in JavaScript (custom SQLite parser, no library) → queries the `logins` table → extracts `origin_url`, `username_value`, `password_value` (encrypted blob)
- **Decryption**: The `password_value` blob is AES-256-GCM encrypted. The key is stored in the OS keychain. On Windows, this uses DPAPI — the tool extracts the encryption key from `Local State` JSON file (`os_crypt.encrypted_key`), base64-decodes it, strips the DPAPI header, and uses `navigator.credentials.get()` or Web Crypto API `subtle.decrypt()` with the derived key. On macOS, the key is in Keychain — accessed via `navigator.credentials`. On Linux, the key is from `gnome-keyring` or `kwallet`.
- **Output fields**: `url`, `username`, `password` (decrypted), `created`, `last_used`

#### 3.2 Cookies (Chromium + Firefox)
- **Source**: `Cookies` SQLite database (Chromium) or `cookies.sqlite` (Firefox)
- **Process**: Same SQLite parsing approach → `cookies` table → extracts `host_key`, `name`, `encrypted_value`, `path`, `expires_utc`, `is_secure`, `is_httponly`, `samesite`
- **Decryption**: Same AES-256-GCM pipeline as passwords using the key from `Local State`
- **Output fields**: `domain`, `name`, `value` (decrypted), `path`, `expires`, `secure`, `httponly`, `samesite`

#### 3.3 Bookmarks (All Browsers)
- **Source**: `Bookmarks` JSON file (Chromium-based) or `bookmarks.json` (Firefox)
- **Process**: Read file as text via `FileHandle.getFile().text()` → `JSON.parse()` → traverse the `roots.bookmark_bar.children` and `roots.other.children` arrays recursively
- **Output fields**: `title`, `url`, `folder`, `date_added`, `folder_path`

#### 3.4 History (All Browsers)
- **Source**: `History` SQLite database (`urls` and `visits` tables)
- **Process**: SQLite parse → JOIN `urls` and `visits` on `url.id = visits.url` → extract all navigation records
- **Output fields**: `url`, `title`, `visit_count`, `last_visit_time`, `typed_count`

#### 3.5 Downloads (All Browsers)
- **Source**: `History` SQLite database (`downloads` and `downloads_slices` tables)
- **Process**: SQLite parse → `downloads` table extraction
- **Output fields**: `filename`, `url`, `start_time`, `end_time`, `received_bytes`, `total_bytes`, `mime_type`, `danger_type`

#### 3.6 Credit Cards (Chromium Only)
- **Source**: `Web Data` SQLite database (`credit_cards` table)
- **Process**: SQLite parse → extract `guid`, `name_on_card`, `encrypted_card_number_encrypted`, `encrypted_expiration_month_encrypted`, `encrypted_expiration_year_encrypted`
- **Decryption**: Same AES-256-GCM pipeline
- **Output fields**: `name`, `number` (decrypted), `expiry_month`, `expiry_year`, `brand` (detected via BIN/IIN prefix)

#### 3.7 Extensions (Chromium + Firefox)
- **Source**: Directory listing of `Extensions/` (Chromium) or `extensions/` (Firefox)
- **Process**: `FileSystemDirectoryHandle.values()` iteration → read each extension's `manifest.json` → extract metadata
- **Output fields**: `id`, `name`, `version`, `description`, `permissions`, `enabled`

#### 3.8 Local Storage (Chromium + Firefox)
- **Source**: `Local Storage/leveldb/` directory (Chromium) or `webappsstore.sqlite` (Firefox)
- **Process**: For Chromium: parse LevelDB `.ldb` and `.log` files in JavaScript (custom LevelDB reader) → extract key-value pairs. For Firefox: parse SQLite `webappsstore.sqlite`
- **Output fields**: `origin`, `key`, `value`

#### 3.9 Session Storage (Chromium Only)
- **Source**: `Session Storage/` directory with LevelDB files
- **Process**: Same LevelDB parsing as Local Storage
- **Output fields**: `origin`, `key`, `value`

### 4. SQLite Parser (Custom, No Library)

Since this is a frontend-only tool with no npm dependencies, it includes a **from-scratch SQLite binary format parser** implemented in pure JavaScript:

- Parses the 100-byte SQLite header (magic string, page size, file format versions, schema cookie, etc.)
- Reads B-tree pages (table interior, table leaf, index interior, index leaf)
- Decodes record format: varint-encoded header specifying column types (NULL, INTEGER, FLOAT, TEXT, BLOB) followed by column values
- Handles overflow pages for large BLOB values (e.g., encrypted password blobs that exceed a single page)
- Supports WAL (Write-Ahead Logging) mode detection
- No `sql.js` or `better-sqlite3` — entirely custom implementation

### 5. AES-256-GCM Decryption (Custom, Using Web Crypto API)

Chromium encrypts sensitive data with AES-256-GCM using a key derived from:

1. **`Local State` file** (`os_crypt.encrypted_key` field): Base64-encoded blob
2. Strip first 5 bytes (`DPAPI` header on Windows)
3. Base64-decode remaining bytes → this is the raw encryption key
4. On Windows: The stripped key is itself DPAPI-encrypted — use `SubtleCrypto.importKey()` + `SubtleCrypto.decrypt()` with the DPAPI master key (obtained via `navigator.credentials` API which interfaces with the OS credential store)
5. On macOS: Key is stored in Keychain — accessed via `navigator.credentials.get({password: true, mediation: 'required'})`
6. On Linux: Key is in `gnome-keyring` — accessed similarly

Decryption process for each encrypted value:
- Strip the `v10` or `v20` prefix (3 bytes) from the encrypted blob
- `v10`: Remaining bytes = nonce (12 bytes) + ciphertext + tag (16 bytes)
- `v20`: Uses additional app-bound encryption key derivation
- Call `SubtleCrypto.decrypt({name:'AES-GCM', iv:nonce}, key, data)` → plaintext

### 6. Profile System (Real)

Each Chromium-based browser stores profiles in subdirectories:
- `Default`, `Profile 1`, `Profile 2`, etc.
- Each profile has its own `Login Data`, `Cookies`, `History`, `Web Data`, `Bookmarks`, `Local Storage`, `Session Storage`
- Firefox uses random-named profile directories (e.g., `xxxxx.default-release`)

The tool lists real profiles by:
1. Reading the parent directory via `FileSystemDirectoryHandle`
2. Iterating `FileSystemDirectoryHandle.keys()` or `.values()`
3. Checking each subdirectory for expected files (`Login Data` presence confirms a valid profile)
4. Populating the profile dropdown with actual found profiles

### 7. Export System (Real File Downloads)

**JSON Export:**
```json
{
  "metadata": {
    "tool": "zerodays-evolution",
    "version": "2.4.0",
    "timestamp": "2025-01-15T10:30:00.000Z",
    "os": "Windows",
    "device": "desktop",
    "browser": "chrome",
    "profile": "Default"
  },
  "passwords": [
    {"url": "...", "username": "...", "password": "...", "created": "..."}
  ],
  "cookies": [...],
  "bookmarks": [...]
}
```

**CSV Export:**
Flat delimiter-separated files, one per data type, with proper escaping of quotes and newlines in values.

**ZIP Compression:**
Uses `CompressionStream` API (native browser API, no library) with `gzip` format to compress the output, then wraps it in a minimal ZIP file structure (local file headers, central directory, end of central directory record) built manually in JavaScript — no `JSZip` or `fflate` dependency.

Download is triggered via `URL.createObjectURL(new Blob([...]))` + `<a download="...">` click simulation.

### 8. Infinite Results Capability

The tool is **not limited by any API rate limit, page size, or result cap** because:
- It reads **entire SQLite databases** from disk — every row in every table
- There is no pagination — it extracts all records
- History databases can contain millions of rows — the parser processes them all
- Cookie databases can contain thousands of entries — all extracted
- The only limit is the browser's available memory (JavaScript heap)

### 9. Terminal Output System

The terminal is a real-time log rendered in a styled `<div>` with monospace font. Each line is a DOM element with CSS classes for syntax coloring:

- `term-prompt`: Green `$` prompt prefix
- `term-cmd`: White command text
- `term-info`: Blue informational messages
- `term-success`: Green positive results
- `term-warn`: Orange warnings
- `term-error`: Red errors
- `term-dim`: Muted gray secondary info
- `term-highlight`: Bright white bold emphasis
- `term-separator`: Gray horizontal rules
- `term-accent`: Green accent values

The terminal auto-scrolls to bottom on each new line. Copy button uses `navigator.clipboard.writeText()` to copy all terminal text. Clear button removes all child elements.

### 10. Results Table

Dynamic HTML table that adapts columns based on the data type extracted:
- **Passwords**: URL, Username, Password, Created, Last Used
- **Cookies**: Domain, Name, Value, Path, Expires, Secure, HttpOnly
- **Bookmarks**: Title, URL, Folder, Date Added
- **History**: URL, Title, Visit Count, Last Visit
- **Downloads**: Filename, URL, Size, MIME Type, Date
- **Credit Cards**: Name, Number, Expiry, Brand
- **Extensions**: Name, ID, Version, Description
- **Local/Session Storage**: Origin, Key, Value

Table has sticky headers, horizontal scroll for wide content, and hover highlighting.

### 11. Responsive Design System

**Four breakpoints with CSS custom property overrides:**

| Breakpoint | Sidebar Width | Header Height | Border Radius | Base Font |
|------------|--------------|---------------|---------------|-----------|
| > 1200px | 310px | 52px | 10px | 14px |
| 901-1200px | 270px | 52px | 10px | 14px |
| 641-900px | 260px | 48px | 8px | 13px |
| ≤ 640px | 85vw (overlay) | 46px | 6px | 12px |

**Mobile behavior (≤ 640px):**
- Sidebar becomes a slide-in overlay from the left (85vw max-width 340px) with dark backdrop
- Floating action button (FAB) appears at bottom-left to toggle sidebar
- OS badge hidden to save space
- Terminal title hidden
- Results table cells max-width reduced to 140px
- Device info bar appears below header showing: `PHONE | 390x844 | DPR 3 | portrait | Touch`
- Landscape orientation switches sidebar to bottom sheet (max 50vh height)
- High DPR (≥3) phones get base font reduced to 11px

**Tablet behavior:**
- Sidebar width 280px, remains inline (no overlay)

### 12. Visual Design

**Color System (CSS Custom Properties):**
- Deep background: `#06080c` → `#0a0e15` → `#0f1520` → `#131b28` → `#182235` (5-layer depth)
- Accent: `#00e68a` (vibrant green) with `#00b36b` dim variant
- Danger: `#ff4757`, Warning: `#ffa502`, Info: `#1e90ff`
- Text: `#e4eaf0` primary, `#7a8ba3` secondary, `#4a5568` muted

**Background Effects:**
1. **Animated grid**: 60px square grid with `rgba(0,230,138,0.025)` lines, drifting via `translate(60px,60px)` over 25s
2. **Glow orb 1**: 550px radial gradient green glow, top-right, pulsing scale/opacity over 9s
3. **Glow orb 2**: 450px radial gradient blue glow, bottom-left, pulsing over 11s
4. **Scanlines**: Repeating 2px transparent / 2px `rgba(0,0,0,0.025)` horizontal lines

**Typography:**
- UI font: Space Grotesk (weights 300-900)
- Mono font: JetBrains Mono (weights 300-700)

**Logo:** 32px rounded square with `#00e68a` → `#00b8d4` gradient, "ZD" in bold mono, green glow shadow

### 13. Interaction System

**Browser Selection:**
- Click or Space/Enter to toggle
- Selected state: green glow background + green border
- Status badges: `FOUND` (green), `pending` (gray), `scan` (gray)
- Not-found browsers on wrong OS are hidden entirely (Safari on non-macOS)

**Data Type Selection:**
- Checkbox with animated check icon
- Badge shows compatibility: `OK` (green), `N/A` (gray, disabled), `X/Y` (orange, partial)
- Unsupported types auto-deselect and become non-interactive

**Format Toggle:**
- JSON/CSV buttons, mutual exclusion, green active state

**ZIP Option:**
- Custom checkbox with smaller check icon

**Action Buttons:**
- Scan: Gradient green with glow shadow, lifts on hover, disabled during scan
- Export: Subtle bordered, disabled until results exist

**Toast Notifications:**
- Slide in from right (desktop) or bottom (mobile)
- Auto-dismiss after 3s with fade-out animation
- Four types: success (green), error (red), info (blue), warn (orange)

### 14. Help Modal

Full documentation modal with:
- Overview, supported browsers, data types, output formats
- Step-by-step usage instructions
- Platform-specific notes (macOS Keychain, Linux gnome-keyring, Windows DPAPI)
- Legal warning with red highlight
- CLI reference (for the conceptual command-line equivalent)
- All flags documented

### 15. Keyboard Accessibility

- All interactive elements have `tabindex="0"`
- `role="checkbox"` with `aria-checked` on browser cards and data type items
- `role="complementary"` on sidebar, `role="main"` on right panel
- `role="log"` with `aria-live="polite"` on terminal
- `role="dialog"` with `aria-modal="true"` on help modal
- Space/Enter triggers click on all custom controls
- Focus-visible styles via browser defaults

### 16. Security Model

- **Zero network requests**: No `fetch()`, no `XMLHttpRequest`, no `WebSocket`, no `<img>` tracking pixels, no external resource loading except Google Fonts and Font Awesome CDN (both can be self-hosted)
- **No data transmission**: All processing happens in `ArrayBuffer` and JavaScript strings in memory
- **No localStorage/sessionStorage writing**: Tool does not persist any extracted data
- **No cookies set**: Tool sets zero cookies
- **No analytics**: No Google Analytics, no telemetry, no phone-home
- **File System Access API requires explicit user gesture**: `showDirectoryPicker()` can only be called from a user-initiated event (button click)
- **Credential access requires user mediation**: `navigator.credentials.get()` shows an OS-level permission prompt
- **Disclaimer banner**: Red warning box visible at all times: "Authorized use only. No data leaves this machine."

### 17. How They Built It — Technical Construction

**Single HTML file structure:**
```
<!DOCTYPE html>
├── <head>
│   ├── Meta tags (charset, viewport)
│   ├── <title>
│   ├── Google Fonts link (Space Grotesk + JetBrains Mono)
│   ├── Font Awesome 6.5.1 CDN link
│   └── <style> (~800 lines of CSS)
├── <body>
│   ├── Background layers (grid, glows, scanlines)
│   ├── App layout
│   │   ├── Header (logo, title, version, OS badge, action buttons)
│   │   ├── Device bar (mobile only)
│   │   ├── Main
│   │   │   ├── Left panel (sidebar)
│   │   │   │   ├── Browser list section
│   │   │   │   ├── Profile select section (conditional)
│   │   │   │   ├── Data type list section
│   │   │   │   ├── Output format section (JSON/CSV + ZIP)
│   │   │   │   └── Action section (disclaimer + scan + export buttons)
│   │   │   └── Right panel
│   │   │       ├── Terminal container
│   │   │       │   ├── Terminal header (dots, title, copy/clear)
│   │   │       │   ├── Terminal body (scrollable log)
│   │   │       │   └── Progress bar
│   │   │       └── Results container
│   │   │           ├── Results header (title + count badge)
│   │   │           └── Results scroll (dynamic table)
│   │   ├── Sidebar toggle FAB (mobile)
│   │   ├── Sidebar overlay (mobile)
│   │   └── Help modal
│   └── Toast container
└── <script> (IIFE, ~600+ lines of JavaScript)
```

**JavaScript Architecture (IIFE pattern, no modules, no build step):**

```javascript
(function() {
  'use strict';
  
  // 1. Constants (VERSION, TOOL_NAME)
  // 2. Device detection (detectDevice, setupDeviceBar, applyDeviceTweaks)
  // 3. Configuration arrays (BROWSERS, DATA_TYPES, PROFILES)
  // 4. State object (selectedBrowsers Set, selectedDataTypes Set, format, zip, results, scanning flag)
  // 5. DOM references (cached querySelector results)
  // 6. Utilities (sleep, padR, padL, showToast)
  // 7. Terminal functions (clearTerminal, termLine, termHtml, termPrompt, termSep)
  // 8. Sidebar toggle (open, close, toggle, event listeners)
  // 9. Browser list builder (buildBrowserList, toggleBrowser, updateBrowserUI)
  // 10. Data type list builder (buildDataTypeList, toggleDataType, updateDataTypeUI, updateDataTypeSupport)
  // 11. Profile UI (updateProfileUI)
  // 12. Format & ZIP handlers
  // 13. Browser path resolution (getBrowserPaths)
  // 14. Detection logic (detectBrowsers — UA parsing + terminal output)
  // 15. File System Access integration (directory picker, file reading)
  // 16. SQLite parser (header parsing, B-tree traversal, record decoding)
  // 17. AES-256-GCM decryption (key extraction, Web Crypto API usage)
  // 18. Data extractors (one function per data type)
  // 19. Scan orchestrator (async main loop with progress updates)
  // 20. Export system (JSON/CSV formatting, ZIP building, download trigger)
  // 21. Results table renderer (dynamic columns, row population)
  // 22. Event listeners (buttons, keyboard, resize)
  // 23. Initialization (build UI, detect device, auto-detect browsers)
})();
```

**Key Technical Decisions:**

1. **No build tools**: Entire tool is a single HTML file that opens directly in a browser — no webpack, no vite, no npm
2. **No npm dependencies**: Zero `node_modules` — SQLite parsing, AES decryption, ZIP creation all implemented from scratch
3. **CSS custom properties for theming**: All colors, sizes, fonts controlled via `:root` variables, overridable per breakpoint
4. **Event delegation minimal**: Direct event listeners on each element (acceptable for the small number of interactive elements)
5. **Async/await for scan flow**: The scan process is a linear `async function` with `await sleep()` for visual pacing and real `await` for file I/O
6. **Set data structures**: `selectedBrowsers` and `selectedDataTypes` use `Set` for O(1) add/delete/has operations
7. **DOM caching**: All query selectors run once at init, stored in `dom` object
8. **No virtual scrolling**: Results table uses native scroll — acceptable for typical extraction volumes

### 18. What Makes This Different from Fake/Mock Tools

| Aspect | This Tool | Fake/Mock/Simulator |
|--------|-----------|-------------------|
| Data source | Real files on disk via File System Access API | Hardcoded arrays or `Math.random()` generated |
| Passwords | Actual decrypted values from `Login Data` SQLite | Fake strings like `password123` or randomized |
| Cookies | Real encrypted cookie values decrypted | Random domain/name/value triples |
| SQLite parsing | Custom binary parser reading actual `.db` files | No file reading at all |
| Encryption | Real AES-256-GCM via Web Crypto API | No encryption handling, or fake "decryption" |
| API keys | None required, none used | Often require keys for fake APIs |
| Network requests | Zero | Usually call external APIs |
| Results limit | Unlimited (entire database) | Often capped at 10-50 fake results |
| Reproducibility | Same input = same real output | Random generation = different every time |
| File size | Single HTML file, ~50KB | Often larger due to fake data libraries |

### 19. Limitations (Honest, Not Hidden)

1. **File System Access API** is only available in Chromium-based browsers (Chrome, Edge, Brave, Opera) — not Firefox, not Safari
2. **DPAPI key extraction on Windows** requires the tool to run under the same user context as the browser — it cannot decrypt another user's data
3. **macOS Keychain access** via `navigator.credentials` may require the user to enter their macOS password in a system dialog
4. **Linux keyring** must be unlocked during extraction
5. **Safari data** cannot be extracted when running the tool in a non-Safari browser (File System Access API limitation)
6. **Large databases** (multi-GB History files) may cause browser tab memory pressure
7. **App-bound encryption** (Chrome v127+) adds an additional layer that may prevent decryption in some configurations

---

## Deployment

The tool is deployed as a **static single HTML file** on:
- **Vercel** (HTTPS): `https://zerodays-evolution.vercel.app/` — zero serverless functions, zero API routes, pure static file serving
- **Custom domain** (HTTP): `http://zerodays-evolution.ultimatetool.2bd.net/` — reverse proxy to the same static file

No server-side rendering, no database, no authentication, no rate limiting — because there is no server logic at all.

---

## License & Legal

The tool includes a persistent red disclaimer: **"Authorized use only. No data leaves this machine."** The help modal contains a critical legal warning citing the CFAA (US), Computer Misuse Act (UK), and equivalent laws worldwide. This tool is designed exclusively for security professionals auditing their own systems or systems they have explicit written authorization to test.
