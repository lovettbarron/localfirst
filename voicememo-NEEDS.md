# VoiceMemo for Ableton Live — Project Needs

A Max for Live audio device that reads Apple Voice Memos from iCloud, lets you select time windows, and pulls them into Live Session clips as WAV files. Purely local — no network calls.

---

## Tools

| Tool | Purpose |
|------|---------|
| **Node 22** | Runtime for all `src/` logic (analysis, windowing, library catalog). Ships with Live 12.4 (`node:sqlite`, `fs`, `path`). |
| **Python 3** | `tools/build-device.py` assembles the `.amxd` binary and generated js files from `src/`. Run via `npm run build`. |
| **`sqlite3`** (macOS) | Read-only queries against `CloudRecordings.db` for memo metadata. |
| **`afconvert`** (macOS) | Converts Voice Memo AAC files to WAV for analysis and audition. |
| **`afinfo`** (macOS) | Reads duration/metadata from source recordings. |
| **Ableton Live 12+** with Max for Live | Host runtime — the device is a `.amxd` loaded on an audio track. |

No npm packages are used. The device bundles only Node and macOS built-ins.

---

## MCP Servers

| Server | Why it helps |
|--------|-------------|
| **filesystem** (local) | The device reads the Voice Memos library (`CloudRecordings.db`, AAC files) and writes WAV/cache to disk. An MCP filesystem server would let pi inspect library state, verify privacy boundaries, and test file I/O without needing Live running. |
| **process / shell** (local) | `build-device.py` calls `python3`, and the device itself invokes `/usr/bin/afconvert`, `/usr/bin/afinfo`, `/usr/bin/sqlite3`. A shell MCP would let pi run the build pipeline and test CLI tools end-to-end. |
| **test-runner** (local) | `npm test` enforces privacy-boundary rules (no network imports, only three files may touch the library). A test MCP could surface these results and track regressions across edits. |

No cloud or remote MCP servers are needed — the project is fully local-first by design.

---

## Skills

| Skill | Reuse value |
|-------|-------------|
| **Privacy-boundary testing** | The `test/privacy-boundary.test.js` enforces that only three files access the Voice Memos library, keeps `PRIVACY:` annotations, and blocks all network imports. This pattern generalizes to any project that touches protected OS resources (contacts, photos, clipboard) and needs auditable boundaries. |
| **Build-script codegen** | `tools/build-device.py` generates self-contained Max patcher JSON from source files, inlining dependencies and asserting no absolute paths leak. The pattern (read sources → emit self-contained artifacts + path assertions) applies to any plugin/device build pipeline. |
| **Segment detection from silence** | The `src/` analysis splits recordings into segments by finding gaps below a configurable dB threshold. This is a general audio pre-processing step useful for any project that needs to chunk recordings (podcast editing, music transcription). |
| **Read-only library access** | The catalog reads `CloudRecordings.db` read-only, falling back to a temp copy when locked. This pattern (read-first, temp-copy-on-contention) generalizes to any SQLite-backed local data access. |
