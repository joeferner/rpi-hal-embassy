# rpi-hal-embassy

[![CI](https://img.shields.io/github/actions/workflow/status/joeferner/rpi-hal-embassy/ci.yml?branch=main&label=CI)](https://github.com/joeferner/rpi-hal-embassy/actions/workflows/ci.yml)
[![crates.io](https://img.shields.io/crates/v/rpi-hal-embassy.svg)](https://crates.io/crates/rpi-hal-embassy)
[![docs.rs](https://img.shields.io/docsrs/rpi-hal-embassy)](https://docs.rs/rpi-hal-embassy)

[Embassy](https://embassy.dev) support for Raspberry Pi boards using the
BCM2836/BCM2837 SoC (Pi 2, Pi 3), on top of the `rpi-hal` crate.

Embassy's executor is portable; what it needs per platform is a time
driver and an architecture-specific idle/wake path. This crate supplies
both:

- an `embassy-time` driver over the BCM System Timer, and
- a thread-mode executor that behaves identically on AArch32 and
  AArch64.

## Status

Both halves are in place: the `embassy-time` driver (`time_driver`) and
the thread-mode executor (`Executor`). `examples/embassy_blink.rs` spawns
two tasks and drives both from `embassy-time` deadlines.

`embassy-executor` compiles for both `armv7a-none-eabi` and
`aarch64-unknown-none-softfloat` with no `platform-*` backend feature,
which is what lets a single executor implementation serve both
architectures.

## Features

- **`usb-host`** (off by default): adds the [`usb`](src/usb.rs) module —
  `Dwc2HostController`, `Dwc2Allocator` and `Dwc2Pipe`, implementing
  `embassy-usb-driver`'s host traits over `rpi-hal`'s DWC2 host
  channels, so [`embassy-usb-host`](https://docs.rs/embassy-usb-host)'s
  enumeration and class drivers run on a Pi. One host channel per pipe,
  taken from the controller's pool; running out is an error rather than
  a queue, so an application that wants more concurrent endpoints than
  the eight this SoC has finds out.

  This is the one adapter here that is *not* a time driver, and it needs
  saying why it isn't in the HAL. `embassy-usb-driver` is trait
  definitions only, like `embassy-net-driver`, whose adapter does live
  there. What differs is that the `UsbPipe` contract is written in
  wall-clock terms — how long to keep retrying a NAK'd control transfer,
  how often to re-poll an interrupt endpoint — while `rpi-hal`'s USB
  layer deliberately has no clock, only bus time (channel-halt and
  start-of-frame interrupts). Supplying the missing clock means
  depending on `embassy-time`, and that is a link-time singleton — the
  thing this crate exists to keep out of the HAL.

  Off by default because the host stack is a large dependency graph that
  an application wanting only the time driver and executor shouldn't
  pay for. Enabling it also turns on `rpi-hal`'s `async` feature, which
  supplies the interrupt-driven transfers underneath.

  The application must route the USB interrupt: `Lic::enable_usb_irq()`,
  and `rpi_hal::usb::dwc2::on_irq()` from its own `__irq_handler`.
  Nothing completes without it, and since `rpi-hal`'s handler is a weak
  no-op, omitting it looks like a hang rather than an error.

  **`embassy-usb-host` 0.1.0 cannot yet drive a Pi**, and the adapter
  here cannot paper over it: the released hub driver matches a hub
  interface on `bInterfaceProtocol == 0x00`, which is a *full-speed* hub
  (USB 2.0 §11.23.1), and every high-speed hub reports `0x01` or `0x02`.
  On this hardware everything is behind the on-board LAN9514, so nothing
  enumerates at all. A second bug routes any high-speed device behind a
  hub as a full-speed split transaction, because `enumerate_port` uses
  the port speed sampled *before* the reset — and high speed is only
  established by the reset handshake.
  [`.cargo/config.toml`](.cargo/config.toml) patches both, plus some
  descriptor-read robustness, from a local checkout while the fixes go
  upstream. The adapter itself is unaffected; this is the stack above
  it.

## Why this isn't a feature of `rpi-hal`

An `embassy-time` driver is installed by *linkage*, not by types.
`embassy_time_driver::time_driver_impl!` defines `#[unsafe(no_mangle)]`
symbols that `embassy-time` declares as `extern "Rust"`, and a program
links only if its crate graph contains exactly one driver.

Behind a cargo feature on the HAL, feature unification means any
dependency that enabled it would force this driver onto the whole
program, and hard-conflict with an application supplying its own. That is
an opt-in that cannot be opted out of. Keeping it in a separate crate
leaves the choice with the application, which is the only place that can
make it. It also keeps the HAL's release cadence independent of the
pre-1.0 Embassy crates.

This is the same split as `esp-hal` / `esp-hal-embassy`. The
`embassy-rp` / `embassy-stm32` model, where the Embassy crate *is* the
HAL, doesn't apply here — `rpi-hal` is the HAL.

## What an application must provide

- **A `critical-section` implementation**, which `rpi-hal`'s `rt` feature
  provides. The driver's timer queue is guarded by one.
- **`rpi-hal`'s `mmu` feature** (enabled by default). The executor's run
  queue uses atomic compare-exchange, and `ldrex`/`strex` are
  UNPREDICTABLE until RAM is mapped as cacheable Normal memory.
- **Interrupt dispatch.** `rpi-hal` leaves `__irq_handler` to the
  application, so this crate can't claim it. The application routes the
  System Timer's interrupt here from its own handler.
- **A linker script**, providing `_start`, `__bss_start` and `__bss_end`
  for `rpi-hal`'s boot code. `rpi-hal` publishes one on the linker search
  path, so the application only has to name it — one `-T` line in
  `.cargo/config.toml`, no script to copy and no build script:

  ```toml
  [target.aarch64-unknown-none-softfloat]
  rustflags = ["-C", "link-arg=-Trpi-link.x"]
  ```
- **A reference to this crate**, even if the application calls nothing
  from it. The time driver is installed by linkage, and a crate nothing
  names is never linked, so the program fails on `_embassy_time_now`
  undefined. Calling `time_driver::init` satisfies this; a program that
  only reads `Instant::now()` needs `use rpi_hal_embassy as _;`.

Neither `rt` nor `mmu` is requested by this crate's own dependency on
`rpi-hal`: which boot sequence and memory setup a program uses is the
application's decision, not a library's to force.

## Wiring it up

```toml
[dependencies]
rpi-hal = "0.1"           # `rt` and `mmu` are on by default
rpi-hal-embassy = "0.1"
embassy-executor = "0.10"
embassy-time = "0.5"      # no `tick-hz-*` feature: this crate pins it
```

```rust
use embassy_time::{Duration, Ticker};
use rpi_hal::{irq, lic::Lic, pac, timer::Timer};
use rpi_hal_embassy::{time_driver, Executor};

#[embassy_executor::task]
async fn heartbeat() {
    let mut ticker = Ticker::every(Duration::from_secs(1));
    loop {
        ticker.next().await;
        // ...
    }
}

#[unsafe(no_mangle)]
pub extern "C" fn kmain() -> ! {
    let peripherals = unsafe { pac::Peripherals::steal() };

    // Three gates, and a deadline fires only with all three open: hand
    // Compare 1 to the driver, unmask interrupts at the CPU, and dispatch
    // the interrupt below.
    let lic = Lic::new(peripherals.LIC);
    time_driver::init(Timer::new(peripherals.SYSTMR), &lic);
    irq::enable_irq();

    // `run` needs `&'static mut`. `static_cell::StaticCell` is the usual
    // way; widening a local's lifetime is sound here only because `kmain`
    // never returns.
    let mut executor = Executor::new();
    let executor = unsafe {
        core::mem::transmute::<&mut Executor, &'static mut Executor>(&mut executor)
    };

    executor.run(|spawner| {
        // The `Result` is on building the task token — a `#[task]` pool
        // already in use — not on spawning it.
        spawner.spawn(heartbeat().unwrap());
    });
}

#[unsafe(no_mangle)]
pub extern "C" fn __irq_handler() {
    let peripherals = unsafe { pac::Peripherals::steal() };

    if Lic::new(peripherals.LIC).is_timer1_pending() {
        // Acknowledges the match itself; nothing else may clear it.
        time_driver::on_timer_irq();
    }
}
```

`unsafe(no_mangle)` is the edition 2024 spelling; on edition 2021 write
`#[no_mangle]`.

An application with other interrupt sources adds them to the same
`__irq_handler`, checking each `Lic::is_*_pending` independently — more
than one can be pending in a single entry.

## Timebase

The System Timer runs at a fixed 1 MHz, so `embassy-time`'s tick rate is
pinned to `tick-hz-1_000_000` here rather than left to the application —
it's a property of the hardware, and an application choosing otherwise
would be choosing a rate the driver cannot deliver.

The ARM generic timer has better deadline primitives (per-core, 64-bit
compare) but is the wrong choice for the global timebase: it runs at
19.2 MHz on this hardware, for which `embassy-time` has no matching
`tick-hz-*` rate, so it would need lossy scaling inside `now()` — the
hottest function on the path. It remains the right primitive for per-core
deadlines if per-core executors are added.

## Building

```sh
cargo build                                         # AArch32 (Pi 2, Pi 3)
cargo build --target aarch64-unknown-none-softfloat # AArch64 (Pi 3)
```

AArch32 is the default target only because it is the one a Pi 2 can run.
Either way the example images link against `rpi-hal`'s published
`rpi-link.x`, which is already at the load address for the target being
built — 0x8000 for a `kernel7.img`, 0x80000 for a `kernel8.img` — so an
image direct-boots where the firmware puts it.

`make pre-commit` runs the whole check set: formatting, clippy, the
library and example builds, and a doc build with warnings denied.

## Examples

See [`examples/`](examples/). Each one's header comment says what it
demonstrates and what its output is evidence for.

`scripts/build-example.sh <name>` produces `target/kernel7.img`;
`scripts/build-example64.sh <name>` produces `target/kernel8.img`.

## License

MIT OR Apache-2.0
