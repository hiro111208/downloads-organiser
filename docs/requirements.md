# Requirements: Downloads Organizer

## 1. Purpose

Downloads Organizer is a CLI tool that scans a Downloads folder, classifies files using configurable rules (primarily file extensions), and organizes them into category-specific directories.

The application must not include a GUI. macOS is the initial target platform. Linux support is required by v0.8 / v1.0.

This document specifies what the product must do. Implementation sequencing follows `plan.md` (v0.1–v1.0).

## 2. Goals

| ID  | Goal                                                                                |
| --- | ----------------------------------------------------------------------------------- |
| G-1 | Users can see how files in Downloads would be classified without changing anything. |
| G-2 | Users can preview the exact destination of each file before any move.               |
| G-3 | Users can organize files into category directories in a single command.             |
| G-4 | The most recent organize operation can be undone.                                   |
| G-5 | Classification rules can be customized via a configuration file.                    |
| G-6 | Newly added files can be organized automatically by watching Downloads.             |
| G-7 | The tool remains usable on large Downloads folders.                                 |
| G-8 | The same project builds and runs on macOS and Linux.                                |

## 3. Non-Goals

| ID   | Out of scope                                                                                                                               |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| NG-1 | Graphical user interface.                                                                                                                  |
| NG-2 | Cloud storage, remote Downloads folders, or network shares as a first-class target.                                                        |
| NG-3 | Content-based classification (MIME sniffing, ML, or reading file contents) in the initial product. Classification is extension/rule based. |
| NG-4 | Recursively reorganizing files that are already inside category subdirectories, unless a later version explicitly adds that behavior.      |
| NG-5 | Docker-based distribution or runtime.                                                                                                      |
| NG-6 | Multi-user or daemon-as-a-service packaging beyond a local `watch` process.                                                                |
| NG-7 | Windows as a supported platform for v1.0. The design should still avoid unnecessary OS-specific code so Windows could be added later.      |

## 4. Users and Environment

- **Primary user:** a person who wants a tidy Downloads folder and is comfortable with a terminal.
- **Initial environment:** macOS, Rust CLI, no GUI, no Docker.
- **Later environment:** Linux, same CLI.

The default scan root is the current user’s Downloads directory for the host OS. The user must be able to override the target directory via a CLI argument.

## 5. Product Versions

Requirements are grouped by the roadmap in `plan.md`. Earlier versions remain required in later versions unless superseded.

| Version | Theme                       | Files modified?                      |
| ------- | --------------------------- | ------------------------------------ |
| v0.1    | Scan and classify           | No                                   |
| v0.2    | Preview destinations        | No                                   |
| v0.3    | Organize (move)             | Yes                                  |
| v0.4    | Undo last organize          | Yes (restore)                        |
| v0.5    | Custom classification rules | Only if organize/watch runs          |
| v0.6    | Auto-organize on new files  | Yes (watch)                          |
| v0.7    | Performance                 | No new user-facing behavior required |
| v0.8    | Cross-platform (Linux)      | Same behavior on Linux as on macOS   |
| v1.0    | Release quality             | Packaging, docs, tests, CLI polish   |

## 6. Functional Requirements

### 6.1 Scan (v0.1)

| ID        | Requirement                                                                                                                                                                                                                                                                                                           |
| --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FR-SCAN-1 | The `scan` command shall enumerate files in the target Downloads directory.                                                                                                                                                                                                                                           |
| FR-SCAN-2 | Scan shall classify each file into exactly one category using the active rule set.                                                                                                                                                                                                                                    |
| FR-SCAN-3 | Scan shall print a summary grouped by category, including a count per category.                                                                                                                                                                                                                                       |
| FR-SCAN-4 | Scan shall not create, move, rename, or delete any files or directories.                                                                                                                                                                                                                                              |
| FR-SCAN-5 | Files that match no rule shall be classified as `Others`.                                                                                                                                                                                                                                                             |
| FR-SCAN-6 | Scan shall skip the category destination directories it would itself create (so already-organized folders are not treated as loose files). Directories that are not category destinations may be ignored or reported as non-file entries; they shall not be moved in later organize steps unless specified otherwise. |

**Default categories** (until custom rules exist in v0.5):

| Category     | Typical extensions (illustrative; implementation may refine)                     |
| ------------ | -------------------------------------------------------------------------------- |
| Images       | `.jpg`, `.jpeg`, `.png`, `.gif`, `.webp`, `.bmp`, `.svg`, `.heic`                |
| Documents    | `.pdf`, `.doc`, `.docx`, `.txt`, `.md`, `.xls`, `.xlsx`, `.ppt`, `.pptx`, `.csv` |
| Videos       | `.mp4`, `.mov`, `.avi`, `.mkv`, `.webm`                                          |
| Archives     | `.zip`, `.rar`, `.7z`, `.tar`, `.gz`, `.tgz`                                     |
| Applications | `.exe`, `.msi`, `.dmg`, `.pkg`, `.app`, `.apk`                                   |
| Others       | everything else                                                                  |

**Example output:**

```text
Images        42
Documents     31
Videos        12
Archives      25
Applications   5
Others        12
```

**Acceptance**

- Running `scan` on a folder with mixed file types prints non-zero counts that match the files present.
- After `scan`, the folder contents are bitwise/path-identical to before the command.

### 6.2 Preview (v0.2)

| ID        | Requirement                                                                                                                                                                                                        |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| FR-PREV-1 | The `preview` command shall list each file that would be moved and its destination path.                                                                                                                           |
| FR-PREV-2 | Destinations shall be under a category directory named after the classification, preserving the original filename unless a collision policy would rename it (preview shall show the name that organize would use). |
| FR-PREV-3 | Preview shall not create, move, rename, or delete any files or directories.                                                                                                                                        |
| FR-PREV-4 | Files that would remain in place (already correctly located, or not eligible) shall either be omitted or clearly marked as no-op; the output must not imply a move that will not happen.                           |

**Example output:**

```text
IMG_001.jpg
  Downloads/
    -> Images/IMG_001.jpg

report.pdf
  Downloads/
    -> Documents/report.pdf
```

**Acceptance**

- Preview destinations match the categories `scan` would assign for the same rule set.
- After `preview`, the folder contents are unchanged.

### 6.3 Organize (v0.3)

| ID       | Requirement                                                                                                                                                                                      |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| FR-ORG-1 | The `organize` command shall create missing category directories under the target folder.                                                                                                        |
| FR-ORG-2 | Organize shall move each eligible file into its category directory, matching the plan shown by `preview` for the same inputs and rules.                                                          |
| FR-ORG-3 | If a destination filename already exists, organize shall not overwrite it. It shall apply a collision strategy (for example, append a numeric suffix) and still complete the move when possible. |
| FR-ORG-4 | If a single file cannot be moved (permissions, locks, missing source, I/O error), organize shall report the error, skip that file, and continue with remaining files.                            |
| FR-ORG-5 | Organize shall print a summary of succeeded moves, skipped files, and failures.                                                                                                                  |
| FR-ORG-6 | Organize shall not recurse into category directories to re-move already organized files.                                                                                                         |
| FR-ORG-7 | Organize should be equivalent to acting on the current preview plan; a user who ran `preview` immediately before `organize` on an unchanged folder shall get the same destinations.              |

**Acceptance**

- After a successful run, classified files exist under the expected category directories and no longer sit loose in Downloads (except `Others` if the product keeps that category as a subdirectory as well — `Others` files shall also be moved into an `Others` directory for consistency).
- Colliding names do not overwrite existing destination files.
- A failure on one file does not abort the entire batch silently.

### 6.4 Undo (v0.4)

| ID        | Requirement                                                                                                                                                                                      |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| FR-UNDO-1 | After a successful (including partial) `organize`, the tool shall persist enough history to reverse those moves.                                                                                 |
| FR-UNDO-2 | The `undo` command shall restore files from the most recent organize operation to their original paths.                                                                                          |
| FR-UNDO-3 | Undo shall only reverse the latest recorded organize (single-level undo), unless a later version documents a history stack.                                                                      |
| FR-UNDO-4 | If a file to restore is missing, undo shall report it and continue restoring the rest.                                                                                                           |
| FR-UNDO-5 | If the original path is occupied, undo shall not overwrite; it shall fail that file with a clear error and continue.                                                                             |
| FR-UNDO-6 | After a successful undo of a move, history for that move shall be treated as consumed so a second `undo` does not replay the same operation.                                                     |
| FR-UNDO-7 | `scan` and `preview` shall not write undo history. `watch`-driven moves (v0.6) shall be undoable under the same rules as `organize`, either as one watch session batch or per documented policy. |

**Acceptance**

- Organize then undo returns files to original locations when those locations are free and files still exist.
- Missing destination files produce warnings, not a crash.

### 6.5 Custom Rules (v0.5)

| ID        | Requirement                                                                                                                                             |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FR-RULE-1 | Classification rules shall be loaded from a user-editable configuration file.                                                                           |
| FR-RULE-2 | Each rule shall map a file extension (or equivalent pattern documented in the config) to a category name.                                               |
| FR-RULE-3 | Built-in defaults shall apply when no config file exists.                                                                                               |
| FR-RULE-4 | User rules shall override defaults for the same extension.                                                                                              |
| FR-RULE-5 | `scan`, `preview`, `organize`, and `watch` shall all use the same active rule set.                                                                      |
| FR-RULE-6 | Invalid configuration (malformed file, empty category, illegal path characters in category names) shall produce a clear error and shall not move files. |

**Example mapping:**

```text
.pdf  -> Documents
.jpg  -> Images
.mp4  -> Videos
.exe  -> Applications
```

**Acceptance**

- Changing the config so `.pdf` maps to `Papers` causes `scan`/`preview`/`organize` to use `Papers` for PDFs.
- A broken config file fails fast with a readable message.

### 6.6 Auto Organize (v0.6)

| ID         | Requirement                                                                                                                                                                                      |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| FR-WATCH-1 | The `watch` command shall monitor the target Downloads directory for newly created (or newly completed) files.                                                                                   |
| FR-WATCH-2 | When a new eligible file appears, watch shall classify it and move it using the same rules as `organize`.                                                                                        |
| FR-WATCH-3 | Watch shall wait until a file is finished writing (not move a still-downloading partial file). The strategy (size-stable delay, ignore known temp extensions, or OS events) shall be documented. |
| FR-WATCH-4 | Watch shall apply the same collision and error handling as `organize`.                                                                                                                           |
| FR-WATCH-5 | Watch shall run until interrupted by the user (for example Ctrl+C) and then exit cleanly.                                                                                                        |
| FR-WATCH-6 | Watch shall log each classification and move (or skip/error) so the user can see what happened.                                                                                                  |

**Flow:**

```text
File created
    |
    v
Classify
    |
    v
Move
```

**Acceptance**

- Copying a completed `.pdf` into Downloads while `watch` is running results in the file appearing under `Documents` (or the configured category) without a separate `organize` command.
- Incomplete downloads are not moved prematurely under the documented wait policy.

### 6.7 Performance (v0.7)

| ID        | Requirement                                                                                                                                                                |
| --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FR-PERF-1 | Scanning and organizing large folders (thousands of files) shall remain practical: no unbounded per-file memory growth beyond what is needed for the current plan/history. |
| FR-PERF-2 | File enumeration and/or independent file moves may use parallel processing where it is safe (no conflicting destinations).                                                 |
| FR-PERF-3 | The project shall include benchmarks for scan and/or organize on a sizable fixture so regressions can be detected.                                                         |

**Acceptance**

- Benchmarks exist and can be run in the repo.
- Parallelism does not introduce overwrite races or lost updates in collision handling.

### 6.8 Cross-Platform (v0.8)

| ID      | Requirement                                                                                                                                                                          |
| ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| FR-OS-1 | OS-specific paths (Downloads location, path separators, reserved names) shall be abstracted so the same crate builds for macOS and Linux.                                            |
| FR-OS-2 | Default Downloads directory resolution shall follow each OS convention (`~/Downloads` on macOS and Linux).                                                                           |
| FR-OS-3 | File moves shall work correctly across the same filesystem; if a cross-device move is required, the tool shall copy-then-delete or fail with a clear error rather than corrupt data. |
| FR-OS-4 | Watch shall function on macOS and Linux, using an appropriate filesystem watcher abstraction.                                                                                        |

**Acceptance**

- CI or documented local checks confirm `cargo build` for macOS and Linux targets.
- Path handling does not assume a single OS (for example, it must not hard-code only macOS layout or Windows drive letters).

### 6.9 Release Quality (v1.0)

| ID       | Requirement                                                                                                                                                                             |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FR-REL-1 | CLI help, subcommand usage, and flags shall be consistent and documented.                                                                                                               |
| FR-REL-2 | User-facing errors shall explain what failed and, when possible, what to do next. They shall not dump raw panics for expected failures (missing folder, bad config, permission denied). |
| FR-REL-3 | Automated tests shall cover classification, preview/organize plan consistency, collision handling, and undo restore/missing-file cases. Details are in Section 12.                      |
| FR-REL-4 | README (and/or man-style docs) shall describe install, commands, config, and safety (`preview` before `organize`).                                                                      |
| FR-REL-5 | Release artifacts shall include macOS and Linux binaries.                                                                                                                               |

## 7. CLI Requirements

Binary name (planned): `download-organizer`.

| Command                       | Required from | Behavior                                               |
| ----------------------------- | ------------- | ------------------------------------------------------ |
| `download-organizer scan`     | v0.1          | Classify and summarize; read-only.                     |
| `download-organizer preview`  | v0.2          | Show per-file destinations; read-only.                 |
| `download-organizer organize` | v0.3          | Create category dirs and move files.                   |
| `download-organizer undo`     | v0.4          | Reverse the last organize (or documented watch batch). |
| `download-organizer watch`    | v0.6          | Monitor and auto-organize new files.                   |

| ID       | Requirement                                                                                                                                                                                                                                                                                                                                                                                                      |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FR-CLI-1 | Each command shall support `--help`.                                                                                                                                                                                                                                                                                                                                                                             |
| FR-CLI-2 | The user shall be able to pass a target directory (for example `--path`) instead of the default Downloads folder.                                                                                                                                                                                                                                                                                                |
| FR-CLI-3 | Exit code `0` means success (including “nothing to do”). Non-zero means the command could not complete its contract (bad args, missing target, invalid config, or organize/undo aborted due to unrecoverable setup errors). Partial per-file failures may still exit `0` if the batch completed with a printed error list, or non-zero if any file failed — the chosen policy must be consistent and documented. |
| FR-CLI-4 | There shall be no GUI entry point.                                                                                                                                                                                                                                                                                                                                                                               |

## 8. Safety and Data Integrity

These are mandatory product principles, not optional polish.

| ID         | Requirement                                                                                                                                   |
| ---------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| NFR-SAFE-1 | `scan` and `preview` never modify the filesystem (aside from reading).                                                                        |
| NFR-SAFE-2 | Users can inspect the proposed plan with `preview` before `organize`.                                                                         |
| NFR-SAFE-3 | Organize never overwrites an existing destination file.                                                                                       |
| NFR-SAFE-4 | Organize is reversible via `undo` for the last recorded operation, within the limits of FR-UNDO-\*.                                           |
| NFR-SAFE-5 | History used for undo shall not store file contents; it stores paths (and any rename applied for collisions).                                 |
| NFR-SAFE-6 | The tool shall not follow a policy of deleting source files except as part of a move (or documented copy-then-delete for cross-device moves). |

## 9. Non-Functional Requirements

| ID    | Area        | Requirement                                                                                                |
| ----- | ----------- | ---------------------------------------------------------------------------------------------------------- |
| NFR-1 | Interface   | CLI only.                                                                                                  |
| NFR-2 | Language    | Implemented in Rust.                                                                                       |
| NFR-3 | Initial OS  | macOS.                                                                                                     |
| NFR-4 | Later OS    | Linux (v0.8+).                                                                                             |
| NFR-5 | Packaging   | Docker is not required.                                                                                    |
| NFR-6 | Performance | Practical on large local Downloads folders; see FR-PERF-\*.                                                |
| NFR-7 | Reliability | Expected I/O and config errors are handled and reported; the process should not panic on routine failures. |
| NFR-8 | Portability | OS-specific filesystem details are isolated behind a small abstraction.                                    |

## 10. Constraints

- No GUI.
- No Docker requirement for development or runtime.
- Initial target OS is macOS. Linux is required from v0.8. Windows is not required for v1.0.
- Classification in the planned product is rule/extension based, not content based.
- Undo is specified as the most recent organization operation, not an unlimited timeline of every historical move.
- Category directories live inside the target folder (for example `Downloads/Images`), not an arbitrary unrelated tree, unless a future flag is added.

## 11. Learning Scope (non-product)

The project is also a Rust learning exercise. Implementation is expected to exercise ownership and borrowing, structs and enums, error handling, file I/O and filesystem APIs, iterators, configuration, serialization, concurrency, and cross-platform development. These are development goals, not user-facing requirements.

## 12. Testing Strategy

This is a filesystem-safety CLI. Tests must prove **no silent mutation**, **preview matches organize**, **never overwrite**, and **undo restores**. Tests are added with each version; they are not deferred to v1.0.

### 12.1 Principles

| ID   | Requirement                                                                                                                                                                                            |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| TS-1 | Tests shall not use the real `~/Downloads` folder. They shall use an isolated temporary directory and the CLI path override (FR-CLI-2).                                                                |
| TS-2 | `scan`, `preview`, and `organize` shall share one planning function so preview/organize consistency (FR-ORG-7) can be tested without duplicating logic.                                                |
| TS-3 | Tests for a version shall land in the same milestone as the feature (v0.1 scan tests with scan, and so on).                                                                                            |
| TS-4 | The default `cargo test` suite shall be deterministic and fast enough for every change. Flaky or slow `watch` tests may be ignored-by-default or gated, with at least one documented integration path. |
| TS-5 | Automated coverage required by FR-REL-3 is the minimum for v1.0: classification, preview/organize plan consistency, collision handling, and undo including missing files.                              |

### 12.2 Layers

Keep production code in testable layers. Prefer many unit and temp-dir tests; keep full CLI and `watch` tests fewer.

| Layer    | Responsibility                         | Test style                           |
| -------- | -------------------------------------- | ------------------------------------ |
| Classify | Extension → category                   | Pure unit tests; no disk             |
| Plan     | Folder listing → `{from, to}` moves    | Temp dirs; no real moves             |
| Execute  | Create dirs, move, collisions, history | Temp dirs; assert the resulting tree |
| CLI      | Args, stdout/stderr, exit codes        | Thin binary tests                    |

Suggested layout:

```text
src/
  classify.rs
  plan.rs
  execute.rs
  history.rs
tests/
  scan.rs
  preview.rs
  organize.rs
  undo.rs
  cli.rs
benches/
  scan.rs
```

**Pyramid:** classify / plan / execute on temp dirs (most tests) → CLI snapshots (fewer) → watch, Criterion, Linux CI (least).

A helper that fingerprints a tree (relative paths → content hashes) should be used to assert read-only commands and undo.

### 12.3 Safety invariants

These tests are mandatory once the related version exists. They implement Section 8 in executable form.

| ID         | Invariant                                                                                           | From                             |
| ---------- | --------------------------------------------------------------------------------------------------- | -------------------------------- |
| TS-SAFE-1  | `scan` and `preview` leave the directory tree (paths and contents) unchanged.                       | FR-SCAN-4, FR-PREV-3, NFR-SAFE-1 |
| TS-SAFE-2  | Each file maps to exactly one category; unmatched extensions are `Others`.                          | FR-SCAN-2, FR-SCAN-5             |
| TS-SAFE-3  | Files already inside category destination directories are not counted or moved again.               | FR-SCAN-6, FR-ORG-6              |
| TS-SAFE-4  | Destinations from `preview` equal destinations used by `organize` for the same tree and rules.      | FR-ORG-2, FR-ORG-7               |
| TS-SAFE-5  | Organize never overwrites an existing destination file; collision policy still moves when possible. | FR-ORG-3, NFR-SAFE-3             |
| TS-SAFE-6  | A failure on one file is reported; remaining files still process.                                   | FR-ORG-4                         |
| TS-SAFE-7  | Organize then undo restores original paths when they are free and files still exist.                | FR-UNDO-2, NFR-SAFE-4            |
| TS-SAFE-8  | A missing file on undo is reported; other files still restore; no panic.                            | FR-UNDO-4                        |
| TS-SAFE-9  | After a successful undo, a second `undo` does not replay the same moves.                            | FR-UNDO-6                        |
| TS-SAFE-10 | Invalid configuration fails before any file is moved.                                               | FR-RULE-6                        |
| TS-SAFE-11 | `scan` and `preview` do not write undo history.                                                     | FR-UNDO-7                        |

### 12.4 Tests by version

| Version             | Required tests                                                                                                                                                                                                                                                                                                                               |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| v0.1 Scan           | Table-driven classification (including case of extensions and no-extension → `Others`). Empty folder, mixed files, folder that already contains category dirs. Scan does not create category directories. Tree unchanged after scan.                                                                                                         |
| v0.2 Preview        | Known fixture → expected destinations (snapshot of CLI output is acceptable). Collision names in preview match the names organize will use. No-op files omitted or marked, never shown as a move. Tree unchanged after preview.                                                                                                              |
| v0.3 Organize       | Happy path: files under category dirs, no loose copies left. Collision does not overwrite. One unreadable or missing file; others still move. Second organize is a no-op (or only leftovers). Preview plan matches organize.                                                                                                                 |
| v0.4 Undo           | Organize then undo restores layout, including collision-renamed names mapped back. Delete a moved file then undo: skip that file, restore the rest. Occupy the original path then undo: no overwrite. Scan/preview do not write history. Second undo is a no-op.                                                                             |
| v0.5 Custom rules   | Missing config → defaults. Override (for example `.pdf` → `Papers`) applies to scan, preview, and organize. Malformed file, empty category, or illegal category path characters → non-zero exit, tree unchanged.                                                                                                                             |
| v0.6 Watch          | Unit-test the “file is finished writing” policy (size-stable delay and/or ignored temp extensions such as `.crdownload`, `.part`, `.download`). One integration test: watch a temp dir, write a completed `.pdf`, assert it moved; write a still-growing file, assert it did not move yet. Isolate if flaky (`#[ignore]` or a feature flag). |
| v0.7 Performance    | Criterion (or equivalent) benches on a generated fixture of thousands of empty files. If moves are parallel, a test that colliding destinations do not race or overwrite.                                                                                                                                                                    |
| v0.8 Cross-platform | CI runs `cargo test` on macOS and Linux. Path logic uses `Path`/`PathBuf`; no hardcoded user home paths.                                                                                                                                                                                                                                     |
| v1.0 Release        | `--help` for every subcommand. Missing target path and invalid config produce non-zero exit and no panic. Exit-code policy (FR-CLI-3) is documented and asserted.                                                                                                                                                                            |

**Rule of thumb for each new command:** (1) pure logic tests for new rules, (2) one temp-dir happy path, (3) one temp-dir dangerous path (overwrite, missing file, bad config, or partial I/O error), (4) one CLI test for stdout/exit code.

### 12.5 Tools

| Tool                          | Use                                                                     |
| ----------------------------- | ----------------------------------------------------------------------- |
| `cargo test`                  | Unit tests next to modules; integration tests under `tests/`            |
| `tempfile`                    | Isolated fake Downloads directories                                     |
| `assert_cmd` and `predicates` | Binary stdout, stderr, and exit codes                                   |
| `insta` (optional)            | Stable `scan` / `preview` text output                                   |
| `proptest` (optional)         | Random names/extensions: still one category; organize never drops bytes |
| `criterion`                   | v0.7 scan/organize benches                                              |

### 12.6 Out of scope for tests

- File contents or MIME types (NG-3). Classification tests use names and extensions only.
- The live user Downloads folder.
- Help-text formatting beyond a smoke `--help`.
- Windows CI for v1.0 (NG-7).
- Running a full `watch` end-to-end on every change if it is flaky; the ready-file policy must still have unit tests.

## 13. Traceability

| Plan item                                        | Requirements                               |
| ------------------------------------------------ | ------------------------------------------ |
| v0.1 Scan                                        | FR-SCAN-\*, FR-CLI (scan), TS-SAFE-1/2/3   |
| v0.2 Preview                                     | FR-PREV-\*, NFR-SAFE-1/2, TS-SAFE-1/4      |
| v0.3 Organize                                    | FR-ORG-\*, NFR-SAFE-3, TS-SAFE-4/5/6       |
| v0.4 Undo                                        | FR-UNDO-\*, NFR-SAFE-4/5, TS-SAFE-7/8/9/11 |
| v0.5 Custom rules                                | FR-RULE-\*, TS-SAFE-10                     |
| v0.6 Auto organize                               | FR-WATCH-\*                                |
| v0.7 Performance                                 | FR-PERF-\*                                 |
| v0.8 Cross platform (Linux in addition to macOS) | FR-OS-\*                                   |
| v1.0 Release                                     | FR-REL-\*, TS-\*                           |
| CLI commands                                     | FR-CLI-\*                                  |
| Safety / undo / CLI-first / cross-platform       | Sections 8–10                              |
| Testing strategy                                 | Section 12                                 |
