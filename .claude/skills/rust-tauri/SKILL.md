---
name: rust-tauri
description: >
  Rust idioms and Tauri v2 patterns for desktop application development. Provides
  expertise in Tauri's command system, capabilities-based security, managed state,
  IPC bridge design, Cargo workspace structuring for testability, streaming XML
  parsing with quick-xml, and typed error handling. Activate this skill when
  designing or implementing Tauri commands, Rust backend logic, IPC data models,
  or Cargo workspace structure.
---

# Rust + Tauri v2 — Desktop Backend

Opinionated, production-tested patterns for the Rust backend of a Tauri v2 application — project structure, capabilities security, IPC design, managed state, XML parsing, and testing strategy.

---

## Tauri v2 vs v1 — Key Differences

Tauri v2 is not backwards-compatible with v1. The most important structural change is the **capabilities system** replacing the v1 `allowlist`.

| Concern | Tauri v1 | Tauri v2 |
| ------- | -------- | -------- |
| Permission model | `tauri.conf.json` `allowlist` object | `src-tauri/capabilities/*.json` files |
| File system access | `allowlist.fs.readFile: true` | `fs:allow-read-files` permission |
| Dialog access | `allowlist.dialog.open: true` | `dialog:open` permission |
| IPC invoke | `tauri.conf.json` `allowlist.invoke` | Always enabled in v2 |
| State management API | `tauri::Manager` trait | Same, but `AppHandle` usage refined |
| JS package | `@tauri-apps/api` v1 | `@tauri-apps/api` v2 (breaking API changes) |

**Do not mix v1 patterns in a v2 project.** The `allowlist` key in `tauri.conf.json` is silently ignored in v2 — permissions that appear to be set via `allowlist` simply have no effect.

---

## Capabilities-Based Security

All permissions are declared in `src-tauri/capabilities/default.json`. Grant the minimum necessary:

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "default",
  "description": "Default permissions for Sonic Ledger",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "fs:allow-read-files",
    "dialog:open"
  ]
}
```

**Key rules:**
- `fs:allow-read-files` permits reading files. It does **not** grant write access.
- Scope the permission to dialog-initiated paths only — never grant a wildcard `fs:scope` unless absolutely necessary.
- `dialog:open` permits opening the OS file picker. `dialog:save` (for write flows) is a separate permission — do not add it until Export is implemented.
- The `"windows": ["main"]` array scopes these permissions to the named window. If you add a second window, it does not inherit permissions automatically.

To add a new capability (e.g., for a future write flow), create a new capability file rather than expanding `default.json`. Capability files compose additively.

---

## Cargo Workspace Structure for Testability

**The single most important architectural decision for a Tauri Rust backend**: separate business logic into a pure library crate, keeping the Tauri binary crate as a thin shell.

```
src-tauri/
├── Cargo.toml          # Workspace root — lists members
├── src/
│   ├── main.rs         # Tauri entry point (bin crate)
│   ├── lib.rs          # Command wiring, app builder
│   └── ...
└── crates/
    └── sonic-ledger-core/
        ├── Cargo.toml  # Library crate — pure Rust, no tauri dep
        └── src/
            ├── lib.rs
            ├── parser.rs
            └── types.rs
```

**Workspace `Cargo.toml`**:
```toml
[workspace]
members = [".", "crates/sonic-ledger-core"]
resolver = "2"
```

**Why this matters**: The top-level `tauri` crate depends on system libraries (`libwebkit2gtk-4.1`, `libdbus-1`, on Linux) that are not present in many CI environments or WSL2. The core crate has no such dependency — it compiles and runs `cargo test` on any platform. This lets you:

- Run unit tests on Linux/WSL2/CI without a full desktop environment
- Test pure Rust logic (parsers, data transforms, business rules) in isolation
- Keep the Tauri binary crate as a thin command-wiring layer with no testable logic of its own

**Rule**: All logic that can be expressed as a pure function belongs in the core crate. The binary crate only wires Tauri commands to core functions and manages Tauri-specific state.

```bash
# Runs on any platform — no system GUI deps required
cargo test -p sonic-ledger-core --manifest-path src-tauri/Cargo.toml

# Runs only on Windows/macOS/Linux with full desktop stack
cargo test --manifest-path src-tauri/Cargo.toml
```

---

## Tauri Command System

### Command Signature

```rust
// src-tauri/src/commands.rs
use tauri::State;
use std::sync::Mutex;

#[tauri::command]
pub fn import_collection(
    path: String,
    state: State<'_, AppState>,
) -> Result<CollectionDto, String> {
    // Business logic delegated to core crate
    let collection = sonic_ledger_core::parser::parse_collection(&path)
        .map_err(|e| e.to_string())?;   // Convert typed error to String at the boundary

    // Store in managed state
    *state.collection.lock().unwrap() = Some(collection.clone());

    Ok(CollectionDto::from(collection))
}
```

**Error conversion at the boundary**: The Tauri command boundary accepts `Result<T, String>`. Convert typed internal errors to strings here — never expose raw Rust error types through IPC. The frontend receives a rejected Promise with the string message.

**Registration in `main.rs`**:
```rust
fn main() {
    tauri::Builder::default()
        .manage(AppState::default())
        .invoke_handler(tauri::generate_handler![
            commands::import_collection,
            commands::get_file_hash,
        ])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### What to Put in a Command vs. the Core Crate

| Belongs in command (`src-tauri/src/`) | Belongs in core crate (`crates/`) |
| ------------------------------------- | --------------------------------- |
| Tauri `State` access | XML parsing |
| `invoke_handler` registration | Data transformation |
| `AppHandle` usage | Hash computation |
| Error-to-String conversion | Domain types |
| DTO construction | Business rules |

---

## Managed State

```rust
// src-tauri/src/state.rs
use std::sync::Mutex;
use sonic_ledger_core::types::Collection;

#[derive(Default)]
pub struct AppState {
    pub collection: Mutex<Option<Collection>>,
}
```

**Patterns:**
- Wrap in `Mutex<Option<T>>` when the state may be absent (not yet loaded). `Option<T>` is the honest representation of "nothing loaded yet" — do not use a sentinel empty struct.
- `.lock().unwrap()` is acceptable when the Mutex cannot be poisoned in normal operation. If a panic could leave the Mutex poisoned, handle `LockResult::Err` explicitly.
- Clear state on error: if `import_collection` fails partway through, reset `state.collection` to `None` before returning `Err` — do not leave stale state.

```rust
// Clear on failure
#[tauri::command]
pub fn import_collection(path: String, state: State<'_, AppState>) -> Result<CollectionDto, String> {
    match sonic_ledger_core::parser::parse_collection(&path) {
        Ok(collection) => {
            *state.collection.lock().unwrap() = Some(collection.clone());
            Ok(CollectionDto::from(collection))
        }
        Err(e) => {
            *state.collection.lock().unwrap() = None; // Always clear on failure
            Err(e.to_string())
        }
    }
}
```

---

## IPC Data Modeling (DTOs)

The Tauri IPC bridge serializes Rust structs to JSON via `serde`. Design DTOs (Data Transfer Objects) for clean frontend consumption:

```rust
// src-tauri/src/dto.rs
use serde::Serialize;

#[derive(Serialize)]
#[serde(rename_all = "camelCase")]
pub struct CollectionDto {
    pub version: Option<String>,
    pub total_tracks: usize,
    pub tracks: Vec<TrackDto>,        // Flat array — frontend builds the Map
    pub playlists: Vec<PlaylistNodeDto>,
}

#[derive(Serialize)]
#[serde(rename_all = "camelCase")]
pub struct TrackDto {
    pub track_id: String,
    pub name: String,
    pub artist: String,
    pub total_time: String,           // "MM:SS" format — frontend parses as needed
    pub average_bpm: Option<String>,  // Option for tracks with no BPM
    pub tonality: Option<String>,     // Rekordbox key field
    pub date_added: Option<String>,   // ISO date string
}
```

**Rules:**
- `#[serde(rename_all = "camelCase")]` on every DTO — TypeScript convention is camelCase; Rust convention is snake_case. The rename prevents the frontend from receiving `total_tracks` (snake) when it expects `totalTracks` (camel).
- Use `Option<T>` for any field that may be absent in the source data — never substitute a sentinel empty string.
- Prefer flat arrays over deeply nested structures for large datasets. The frontend can reconstruct a `Map<id, T>` from a flat array in O(n) — sending a deeply nested Rust tree adds serialization overhead and complicates the TypeScript types.
- Keep DTOs as pure `#[derive(Serialize)]` structs. Do not implement business logic on DTOs; put it on the domain types in the core crate.

---

## Typed Error Enums

Never use `String` as an error type inside the core crate. Use a typed enum:

```rust
// crates/sonic-ledger-core/src/parser.rs
use std::io;

#[derive(Debug)]
pub enum ParseError {
    WrongRootElement,
    MissingCollectionSection,
    Io(io::Error),
    Xml(quick_xml::Error),
}

impl std::fmt::Display for ParseError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Self::WrongRootElement =>
                write!(f, "Not a Rekordbox XML file — expected <DJ_PLAYLISTS> root element"),
            Self::MissingCollectionSection =>
                write!(f, "Rekordbox XML is missing the <COLLECTION> section"),
            Self::Io(e) =>
                write!(f, "Could not read file: {e}"),
            Self::Xml(e) =>
                write!(f, "XML parse error: {e}"),
        }
    }
}

impl From<io::Error> for ParseError {
    fn from(e: io::Error) -> Self { Self::Io(e) }
}

impl From<quick_xml::Error> for ParseError {
    fn from(e: quick_xml::Error) -> Self { Self::Xml(e) }
}
```

The `From` implementations allow `?` to auto-convert IO and XML errors — no manual `.map_err()` needed in parser code. The `Display` impl produces the human-readable string that eventually reaches the frontend via `.to_string()` at the command boundary.

---

## Streaming XML Parsing with `quick-xml`

For large XML files (50–150 MB Rekordbox exports), **always use streaming SAX-style parsing** — never deserialize the full document into a DOM tree. `quick-xml`'s `Reader` is the right tool:

```rust
use quick_xml::Reader;
use quick_xml::events::Event;
use std::io::BufReader;
use std::fs::File;

pub fn parse_collection(path: &str) -> Result<Collection, ParseError> {
    let file = File::open(path)?;     // io::Error auto-converts via From impl
    let buf_reader = BufReader::new(file);
    let mut reader = Reader::from_reader(buf_reader);
    reader.config_mut().trim_text(true);

    let mut buf = Vec::new();
    let mut version: Option<String> = None;
    let mut tracks: HashMap<String, Track> = HashMap::new();
    let mut in_collection = false;

    loop {
        match reader.read_event_into(&mut buf)? {  // quick_xml::Error auto-converts
            Event::Start(ref e) => {
                match e.name().as_ref() {
                    b"DJ_PLAYLISTS" => {
                        // Read the Version attribute — absent is not an error
                        version = e.attributes()
                            .filter_map(|a| a.ok())
                            .find(|a| a.key.as_ref() == b"Version")
                            .and_then(|a| String::from_utf8(a.value.to_vec()).ok());

                        // Validate root element
                    }
                    b"COLLECTION" => { in_collection = true; }
                    b"TRACK" if in_collection => {
                        // Parse track attributes
                        let track = parse_track_element(e)?;
                        tracks.insert(track.track_id.clone(), track);
                    }
                    _ => {}
                }
            }
            Event::End(ref e) => {
                if e.name().as_ref() == b"COLLECTION" { in_collection = false; }
            }
            Event::Eof => break,
            _ => {}
        }
        buf.clear(); // Must clear the buffer after every event
    }

    Ok(Collection { version, tracks, playlists: vec![] })
}
```

**Critical detail**: `buf.clear()` must be called after every event. Forgetting it causes the buffer to grow unboundedly — for a 30k-track file this eventually causes an OOM panic.

**Skipping unknown elements gracefully**: The `_ => {}` arm handles any element name not explicitly matched. `quick-xml` events include all element types (cue data, memory cues, position marks in Rekordbox 6.x) — unknown elements are silently skipped without error.

---

## SHA-256 File Hashing

Use `sha2` + `hex` for file integrity checks:

```rust
use sha2::{Sha256, Digest};
use std::io::{BufReader, Read};
use std::fs::File;

pub fn hash_file(path: &str) -> Result<String, std::io::Error> {
    let file = File::open(path)?;
    let mut reader = BufReader::new(file);
    let mut hasher = Sha256::new();
    let mut buffer = [0u8; 8192];

    loop {
        let n = reader.read(&mut buffer)?;
        if n == 0 { break; }
        hasher.update(&buffer[..n]);
    }

    Ok(hex::encode(hasher.finalize()))
}
```

Read in 8 KB chunks — never read the entire file into memory for hashing. This keeps memory usage flat regardless of file size.

---

## Testing Strategy

### Unit Tests in the Core Crate

Test pure Rust functions using inline `#[cfg(test)]` modules. Use small inline XML strings as fixtures — do not commit large binary XML files:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    const MINIMAL_5X: &str = r#"<?xml version="1.0" encoding="UTF-8"?>
<DJ_PLAYLISTS Version="5.6.0">
  <COLLECTION Entries="2">
    <TRACK TrackID="1" Name="Track One" Artist="Artist A"
           TotalTime="210" AverageBpm="128.0" Tonality="Cmaj"
           DateAdded="2024-01-01" />
    <TRACK TrackID="2" Name="Track Two" Artist="Artist B"
           TotalTime="190" AverageBpm="140.0" Tonality="Amin"
           DateAdded="2024-01-02" />
  </COLLECTION>
  <PLAYLISTS>
    <NODE Type="0" Name="ROOT">
      <NODE Type="1" Name="My Playlist" KeyType="0" Entries="2">
        <TRACK Key="1"/>
        <TRACK Key="2"/>
      </NODE>
    </NODE>
  </PLAYLISTS>
</DJ_PLAYLISTS>"#;

    #[test]
    fn parses_5x_collection() {
        let result = parse_collection_from_str(MINIMAL_5X);
        assert!(result.is_ok());
        let c = result.unwrap();
        assert_eq!(c.tracks.len(), 2);
        assert_eq!(c.version.as_deref(), Some("5.6.0"));
    }

    #[test]
    fn version_absent_is_not_error() {
        let xml = r#"<DJ_PLAYLISTS><COLLECTION Entries="0"></COLLECTION><PLAYLISTS></PLAYLISTS></DJ_PLAYLISTS>"#;
        let result = parse_collection_from_str(xml);
        assert!(result.is_ok());
        assert!(result.unwrap().version.is_none());
    }

    #[test]
    fn wrong_root_element_returns_error() {
        let xml = r#"<FOO><BAR/></FOO>"#;
        let result = parse_collection_from_str(xml);
        assert!(matches!(result.unwrap_err(), ParseError::WrongRootElement));
    }
}
```

Expose a `parse_collection_from_str(xml: &str)` function alongside `parse_collection(path: &str)` — the string version is trivially testable without a filesystem, while the path version is what commands call.

### Integration Tests

Place integration tests in `src-tauri/tests/`. Generate large fixtures programmatically — never commit them:

```rust
// src-tauri/tests/parser_perf.rs
#[test]
fn parse_30k_tracks_under_3_seconds() {
    let xml = generate_tracks_xml(30_000); // helper that builds the XML string
    let start = std::time::Instant::now();
    let result = sonic_ledger_core::parser::parse_collection_from_str(&xml);
    let elapsed = start.elapsed();

    assert!(result.is_ok());
    assert_eq!(result.unwrap().tracks.len(), 30_000);
    assert!(elapsed.as_secs() < 3, "Parse took {elapsed:?}");
}
```

### Avoiding the Tauri Runtime in Tests

The Tauri runtime (`AppHandle`, `State`, `Manager`) requires a running application and cannot be easily constructed in a test. The workspace crate pattern solves this: business logic lives in the core crate (no Tauri dependency), so it is always testable in isolation. The command layer (which does depend on Tauri) is kept thin enough to not need testing — its correctness follows from the core crate tests.

If you must test a Tauri command directly, extract its logic into a free function that takes plain Rust types, and test the free function:

```rust
// Testable free function in core crate
pub fn do_import(path: &str) -> Result<Collection, ParseError> { ... }

// Thin Tauri command — just wires types, not testable in isolation
#[tauri::command]
pub fn import_collection(path: String, state: State<'_, AppState>) -> Result<CollectionDto, String> {
    let collection = sonic_ledger_core::do_import(&path).map_err(|e| e.to_string())?;
    *state.collection.lock().unwrap() = Some(collection.clone());
    Ok(CollectionDto::from(collection))
}
```

---

## Common Pitfalls

**`tauri.conf.json` `allowlist` key has no effect in Tauri v2**
Permissions must be in `src-tauri/capabilities/*.json`. The `allowlist` key is a v1 concept and is silently ignored in v2.

**Forgetting `buf.clear()` in the `quick-xml` event loop**
The buffer grows indefinitely. For large files this is a slow memory leak that eventually OOMs. Always call `buf.clear()` after each `read_event_into` call.

**`Mutex` poisoning on panic**
If code inside a `lock()` guard panics, the Mutex is poisoned and all subsequent `.lock().unwrap()` calls on the same Mutex will also panic. Avoid panicking inside a Mutex guard. For truly unrecoverable states, let the Tauri process restart.

**`serde` snake_case fields reaching TypeScript as snake_case**
Add `#[serde(rename_all = "camelCase")]` to all DTO structs. Without it, `total_tracks` arrives as `total_tracks` in JavaScript, which is unexpected and will cause TypeScript type errors if the interface is correctly typed as `totalTracks`.

**WSL2: `cargo test` at workspace root fails with linker errors**
The top-level Tauri crate requires system GUI libraries not present in WSL2. Run `cargo test -p <core-crate>` to test only the library crate. The full binary is built and tested on Windows.

**`File::open` keeps the file handle open across async boundaries**
`quick-xml`'s `Reader` holds a reference to the reader. In synchronous Tauri commands, this is fine — the file handle is closed when the `Reader` drops at the end of the function. Never `.await` while holding the file handle open.
