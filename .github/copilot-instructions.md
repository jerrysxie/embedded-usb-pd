# Copilot Instructions — embedded-usb-pd

## Build, Test, and Lint

```bash
# Build (host)
cargo build
cargo build --all-features

# Build for embedded target (no-std)
cargo check --target thumbv8m.main-none-eabihf --no-default-features

# Run all tests (CI uses cargo-hack to test all feature combinations)
cargo hack --feature-powerset --exclude-features defmt test --all-targets --locked

# Run a single test
cargo test <test_name>

# Lint
cargo clippy --all-features
cargo fmt --check

# Check all feature combinations compile
cargo hack --feature-powerset check

# Dependency checks (license, advisories, bans)
cargo deny check --all-features
```

## Architecture

This is a `#![no_std]` Rust library providing common types for USB Power Delivery (USB PD) and USB Type-C Connector System Software Interface (UCSI). It is a **types-only crate** — it contains no hardware drivers, I/O, or async runtime code. Downstream driver crates (e.g., tps6699x) depend on these types.

### Module overview

| Module | Purpose |
|--------|---------|
| `pdo` | Power Data Object types — source PDOs, sink PDOs, RDOs, and the `Contract` type. Covers Fixed, Battery, Variable, and Augmented (SPR PPS, EPR AVS) PDO kinds. |
| `ado` | Alert Data Object types (OCP, OTP, OVP, battery status change, power button events, etc.) |
| `ucsi` | UCSI command types, opcodes, and serialization via `bincode` |
| `ucsi::cci` | Command Completion Indicator — bitfield wrapper for the UCSI CCI register |
| `ucsi::lpm` | LPM (Local Policy Manager) command data types — per-connector UCSI commands like `GetConnectorStatus`, `SetUor`, `GetPdos`, etc. |
| `ucsi::ppm` | PPM (Platform Policy Manager) types — includes the PPM state machine (`StateMachine`), PPM-level commands (`AckCcCi`, `SetNotificationEnable`, `GetCapability`), and reset/cancel logic |
| `vdm` | Vendor Defined Message types — SVDM header commands, SVIDs, and alt mode IDs |
| `type_c` | Type-C current levels and connection state |
| `pdinfo` | PD controller info types — power path status, alternate mode status (DisplayPort, Thunderbolt, USB4) |
| `constants` | USB PD spec timing constants (e.g., `tPSTransition`) using typed `Range<T>` wrappers |

### Key design patterns

- **Bitfield structs**: Raw protocol fields are defined with the `bitfield` crate. Public API types wrap these with a typed outer struct that exposes `bool`/enum accessors and implements `Debug` and (optionally) `defmt::Format` manually. See `ado.rs`, `ucsi/cci.rs`, and `pdinfo.rs` for examples.
- **PDO Raw/Data split**: Each PDO type has a raw bitfield struct for wire format and a `Data` struct with typed fields (voltages in mV, currents in mA). Conversion between them uses `From`/`TryFrom` traits.
- **Port ID newtypes**: `LocalPortId` (per-controller port) and `GlobalPortId` (system-wide unique port) are `#[repr(transparent)]` newtypes over `u8`, both implementing the `PortId` trait. Many UCSI types are generic over `T: PortId`.
- **Serialization**: UCSI command data is serialized/deserialized with `bincode` (fixed int encoding, little-endian, `no_std` compatible). Types implement `bincode::Encode` and `bincode::Decode`.
- **State machine**: The PPM state machine in `ucsi::ppm::state_machine` models UCSI spec section 6.1 with explicit `State`, `Input`, and `Output` enums. It includes a Mermaid diagram rendered via `aquamarine`.

### Feature flags

| Feature | Purpose |
|---------|---------|
| `defmt` | Embedded debug logging — adds `defmt::Format` derives to all public types |

When `defmt` is enabled, the `PortId` trait additionally requires `defmt::Format`. All public types use `#[cfg_attr(feature = "defmt", derive(defmt::Format))]`.

## Key Conventions

### Strict clippy policy

The crate denies `unwrap_used`, `indexing_slicing`, `panic`, `unreachable`, `todo`, and `unimplemented` (see `[lints.clippy]` in `Cargo.toml`). Use `.get()` for fallible indexing and propagate errors with `?` instead of panicking. When a match is provably exhaustive due to a bitmask, use `#[allow(clippy::unreachable)]` with a panic safety comment.

### Error types

- `Error<BE>` — top-level error type, wraps either a bus error (`BE`) or a `PdError`
- `PdError` — protocol-level errors (`InvalidParams`, `Busy`, `Failed`, `Timeout`, etc.)
- Per-module invalid-conversion error types (e.g., `ado::InvalidType`, `pdo::ExpectedPdo`, `pdo::InvalidApdoKind`) that convert into `PdError` via `From`

### Testing patterns

Tests are colocated in `#[cfg(test)] mod test` (or `mod tests`) blocks within their source files. Tests exercise:
- Round-trip encoding/decoding (e.g., `u32` ↔ ADO, PDO ↔ raw bitfield)
- Boundary/error cases for `TryFrom` conversions
- State machine transitions and invalid transitions

No external test harness or mocking framework is used — tests operate purely on type conversions and logic.

### Formatting

`rustfmt.toml` enforces: `group_imports = "StdExternalCrate"`, `imports_granularity = "Module"`, `max_width = 120`.

### Code review

When reviewing pull requests:

- **Do not** comment on formatting, style issues, or compilation errors — these are enforced by `cargo fmt`, `cargo clippy`, and `cargo check` in CI.
- **Do** flag if a CI workflow is not covering a feature (e.g., a new feature flag missing from `cargo-hack` powerset checks or `cargo-deny` runs).

### PR process

- Create draft PRs first; ensure CI passes before requesting review
- Maintain clean commit history (each commit builds without warnings, squash fixup commits)
- Commits are not squash-merged; rebase to keep history clean

### Commit messages

- Subject line: ~50 characters max, capitalized, imperative mood (e.g., "Fix bug" not "Fixed bug"), no trailing period
- Separate subject from body with a blank line
- Wrap body at 72 characters
- Use the body to explain *what* and *why*, not *how*
