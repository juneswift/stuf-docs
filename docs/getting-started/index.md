!!! warning "Early-stage software"
    STUF is not production-ready. It has not undergone independent security review and should not be used in production update paths without one. See the [GitHub repo](https://github.com/juneswift/stuf) for current status.

# How to use STUF

STUF is organized as a Rust workspace of composable crates. Core pieces are imported as normal Rust crates, while optional environment bindings and profile choices are controlled through Cargo feature flags where appropriate. The goal is to keep the verification surface explicit and avoid pulling unnecessary components into constrained builds.

This section covers getting the code running locally, understanding the workspace structure, and how the key concepts fit together.

## Quick start

You need a Rust toolchain. If you don't have one, install it from [rustup.rs](https://rustup.rs).

```bash
git clone https://github.com/juneswift/stuf.git
cd stuf
cargo test
```

All tests should pass. If anything fails, open an issue.

## Workspace layout

STUF is a Cargo workspace with five crates, each with a distinct responsibility.

```
stuf-core       # trust kernel — verified types, verifier traits, error model
stuf-encoding   # canonical encoding — deterministic serialization, hash inputs
stuf-env        # environment bindings — crypto, transport, clock, storage
stuf-protocols  # protocol profiles — TUF role chain, metadata verification
stuf-examples   # working examples — publisher fixture, embedded toaster demo
```

The dependency direction flows one way: `stuf-protocols` depends on `stuf-core` and `stuf-env`, not the other way around. The kernel has no knowledge of protocols.

### stuf-core

The trust kernel is the smallest part of STUF. It defines the types that make trust transitions explicit in code:

```rust
// Unverified<T> and Verified<T> are different types.
// The compiler prevents you from using one where the other is expected.
pub struct Verified<T> { inner: T }

pub trait Verifier {
    fn verify(&self, input: Unverified<T>) -> Result<Verified<T>, Error>;
}
```

The kernel owns no transport, no filesystem access, no clock, and no protocol-specific policy. It defines the simplest vocabulary that everything else uses.

### stuf-encoding

Canonical encoding ensures that signature inputs are deterministic: the same metadata always produces the same bytes, regardless of which implementation serialized it. This matters because signatures are computed over the canonical form, not the wire form.

### stuf-env

Environment bindings are composable. `stuf-env` implements the traits defined in `stuf-core` behind Cargo feature flags:

```toml
[dependencies.stuf-env]
version = "0.1"
default-features = false
features = [
  "crypto-ed25519",    # Ed25519 signature verification
  "transport-uart",    # UART transport for embedded
  "clock-rtos",        # RTOS clock binding
  "encoding-cbor",     # CBOR encoding for constrained devices
]
```

On a cloud target you might use `crypto-ring`, `transport-http`, and `clock-std`. On a bare-metal microcontroller you might use `crypto-tinycrypt`, `transport-uart`, and `clock-fixed`. The trust kernel and verification logic are identical, only the environment bindings change.

### stuf-protocols

Protocol profiles implement the full verification chain for a specific security protocol. The TUF profile verifies the Root → Targets → Snapshot → Timestamp chain, checks role thresholds, validates expiry, and authorizes the target artifact.

Protocol profiles are deliberately separate from the kernel.

### stuf-examples

The examples are the fastest way to see how everything fits together. The publisher example generates a signed metadata tree. The embedded toaster demo runs the full verification flow on an ARM Cortex-M3 target in QEMU — no heap, no allocator, root of trust baked in at compile time.

## Running the tests

```bash
# Run everything
cargo test

# Run a specific crate
cargo test -p stuf-core
cargo test -p stuf-encoding
cargo test -p stuf-env
cargo test -p stuf-protocols

# Check the no-default-features build, important for embedded targets
cargo test --no-default-features

# Full development check
cargo fmt && cargo clippy --all-targets --all-features && cargo test
```

## Running the examples

The publisher example generates a signed metadata tree you can use to test verification:

```bash
cargo run -p stuf-examples --bin publisher
```

The embedded example targets ARM Cortex-M3. Install the cross-compilation target first:

```bash
rustup target add thumbv7m-none-eabi
cargo build -p stuf-examples --target thumbv7m-none-eabi
```

To run it under QEMU:

```bash
cargo run -p stuf-examples --target thumbv7m-none-eabi
```

This runs the full verification flow — fetching metadata over semihosting, verifying the trust chain, and checking the target artifact — on a simulated embedded device with no heap allocation.