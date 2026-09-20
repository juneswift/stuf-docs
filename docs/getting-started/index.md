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

STUF is a Cargo workspace organized around five main components, each with a distinct responsibility.

```text
stuf-core       # trust kernel — verified types, verifier traits, error model
stuf-encoding   # canonical encoding — deterministic serialization, hash inputs
stuf-env        # environment bindings — crypto, transport, clock, storage
stuf-protocols  # protocol profiles — TUF role chain, metadata verification
stuf-examples   # working examples — publisher fixture, embedded toaster demos
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

Protocol profiles implement the full verification chain for a specific security protocol. The TUF profile verifies the Root → Timestamp → Snapshot → Targets chain, checks role thresholds, validates expiry, and authorizes the target artifact.

Protocol profiles are deliberately separate from the kernel.

### stuf-examples

The examples are the fastest way to see how everything fits together. The publisher example generates a signed metadata tree and sample firmware artifact. The standard toaster demo runs the full verification flow on an ARM Cortex-M3 target in QEMU using a small allocator. A separate `toaster-no-heap` example runs the same verification flow with fixed buffers and no global allocator. In both cases, the root of trust is baked in at compile time to model manufacture-time provisioning.

## Running the tests

```bash
# Run everything
cargo test

# Run a specific crate
cargo test -p stuf-core
cargo test -p stuf-encoding
cargo test -p stuf-env
cargo test -p stuf-tuf

# Check the no-default-features build, important for embedded targets
cargo test --no-default-features

# Full development check
cargo fmt && cargo clippy --all-targets --all-features && cargo test
```

## Running the examples

First generate the signed TUF repository and sample firmware artifact:

```bash
cargo run -p publisher
```

The publisher writes the repository to:

```text
stuf-examples/.generated/publisher-repo/
```

and copies the trusted root metadata into the toaster factory directory to model provisioning at manufacture time.

The embedded examples target ARM Cortex-M3. Install the cross-compilation target:

```bash
rustup target add thumbv7m-none-eabi
```

The demos run under QEMU using the `lm3s6965evb` Cortex-M3 machine. On macOS:

```bash
brew install qemu
```

You can verify QEMU is available with:

```bash
qemu-system-arm --version
```

### Standard toaster

Build the optimized embedded binary:

```bash
cargo build -p toaster --target thumbv7m-none-eabi --release
```

The release build is required because the unoptimized debug build exceeds the 256 KB flash budget of the emulated target.

Run the toaster under QEMU:

```bash
qemu-system-arm \
  -M lm3s6965evb \
  -cpu cortex-m3 \
  -nographic \
  -semihosting-config enable=on,target=native \
  -kernel target/thumbv7m-none-eabi/release/toaster
```

The demo starts at firmware `v1.0.0`, verifies the TUF metadata chain and the `v1.1.0` firmware artifact, and then performs a simulated flash installation.

### No-heap toaster

Build the no-heap variant:

```bash
cargo build -p toaster-no-heap --target thumbv7m-none-eabi --release
```

Run it under QEMU:

```bash
qemu-system-arm \
  -M lm3s6965evb \
  -cpu cortex-m3 \
  -nographic \
  -semihosting-config enable=on,target=native \
  -kernel target/thumbv7m-none-eabi/release/toaster-no-heap
```

The `toaster-no-heap` profile performs the same verification flow using fixed buffers and no global allocator.

Both demos execute STUF verification logic as ARM Cortex-M3 binaries under QEMU. They fetch metadata through semihosting, verify the TUF trust chain, verify the target firmware artifact, and perform a simulated flash write. The demo does not yet reboot into a second firmware image.
