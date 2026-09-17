# Downloads Organizer

A CLI-based Downloads folder organizer written in Rust.

Downloads Organizer scans files in the Downloads folder, classifies them based on configurable rules such as file extensions, and organizes them into category-specific directories.

The application is designed as a CLI tool without a GUI.

## Development

Initial target is **macOS**. Linux is planned later. Docker is not required. There is no GUI.

### Prerequisites

- [Rust](https://rustup.rs/) via `rustup` (stable toolchain)
- A terminal

Check that the toolchain is available:

```bash
rustc --version
cargo --version
```

### Setup

The repo is already a Cargo binary crate (`download-organizer`). From the project root:

```bash
cargo build
```

### Run

```bash
cargo run
```

Pass CLI arguments after `--`:

```bash
cargo run -- scan --path /path/to/folder
```

Do not point `organize` or `watch` at your real Downloads folder until those commands are implemented and you have previewed the plan. Tests and local experiments should use a temporary directory.

Release binary (after `cargo build --release`):

```bash
./target/release/download-organizer
```

### Test

```bash
cargo test
```

See `docs/requirements.md` (Section 12) for the testing strategy.

### Format and lint (optional)

```bash
cargo fmt
cargo clippy
```

## CLI

Planned commands:

```bash
download-organizer scan
download-organizer preview
download-organizer organize
download-organizer undo
download-organizer watch
```
