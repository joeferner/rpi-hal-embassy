# Changelog

Notable changes to `rpi-hal-embassy`, in the format of
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/). This crate
follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- A `multicore` feature, for running an executor on cores 1-3 as well as
  core 0, and an `embassy_multicore` example demonstrating one per core.
  The feature adds no code: it selects `rpi-hal`'s cross-core
  `critical-section` implementation, which the driver's shared timer
  queue needs once a second core can reach it.
- An `embassy-net-driver` feature and a `lan9514` module: the
  `embassy-net` adapter over `rpi-hal`'s LAN9514 USB-Ethernet driver,
  moved here from `rpi-hal` (where it was that crate's
  `embassy-net-driver` feature) and rebuilt on
  `embassy-net-driver-channel`.

  `lan9514::new` returns a `Lan9514Driver` for `embassy_net::new` and a
  `Lan9514Runner` the application spawns as a task. The split is what
  makes an async driver possible at all: `embassy_net_driver::Driver`'s
  `receive`/`transmit` are synchronous, so a `Driver` implemented
  directly over the chip has nowhere to await and must do its USB work
  with the blocking methods, holding the executor for the length of every
  transfer. As a queue pair plus a task, the USB work is free to await.

  Two consequences for an application. It must dispatch the USB interrupt
  (`Lic::enable_usb_irq`, and `usb::dwc2::on_irq` from its
  `__irq_handler`) — the runner's transfers complete on that and nothing
  else. And it no longer calls `lan9514::wake_rx` on a ticker, because
  there is nothing left to poll: the receive parks on the bulk endpoint
  and wakes on the interrupt, so the latency-versus-wake-ups interval an
  application used to have to choose is gone.

  The runner takes two USB host channels rather than one, one per
  direction, so a transmit never has to cancel a parked receive.

- **A USB host controller for `embassy-usb-host`** (`usb-host` feature,
  off by default): `usb::Dwc2HostController`, `usb::Dwc2Allocator` and
  `usb::Dwc2Pipe`, implementing `embassy-usb-driver`'s host traits over
  `rpi-hal`'s DWC2 host channels. With it, `embassy-usb-host`'s
  enumeration and class drivers run on a Pi: one host channel per pipe,
  drawn from the controller's pool and reported as
  `HostError::OutOfPipes` when exhausted rather than queued.

  This adapter sits here rather than in the HAL beside the
  `embassy-net-driver` one — even though `embassy-usb-driver` is
  likewise trait definitions only — because the `UsbPipe` contract is
  written in wall-clock terms (a control transfer's NAK-retry budget, an
  interrupt endpoint's `bInterval`) and `rpi-hal`'s USB layer
  deliberately has only bus time. Supplying the clock means depending on
  `embassy-time`, a link-time singleton, which is the thing this crate
  exists to keep out of the HAL.

  Known limits, all documented on the module: a control data stage is
  capped at 256 bytes (the HAL's per-channel DMA buffer), a bulk buffer
  must be cache-line aligned and a whole number of cache lines long,
  isochronous transfers are unimplemented, and overcurrent is never
  reported.
- `examples/embassy_usb_keyboard.rs`, reading a USB keyboard through the
  full `embassy-usb-host` stack — one task servicing the on-board hub's
  status-change endpoint, another polling the keyboard, each on its own
  host channel.

  Running it needs `embassy-usb-host` fixes that are not released yet,
  patched in from a local checkout by `.cargo/config.toml`. The released
  0.1.0 registers no high-speed hub at all (its hub driver matches only
  `bInterfaceProtocol == 0x00`, a full-speed hub) and misroutes every
  high-speed device behind a hub as a full-speed split. `README.md` has
  the detail. None of it is in this crate's own code.

  That checkout now reaches the network side too, which is worth knowing
  before a release rather than after: patching `embassy-sync` for the USB
  stack leaves the released `embassy-net-driver-channel` unable to compile
  against it, so the channel crate comes from the checkout as well and
  `lan9514` is written against its slot-based runner API (`RxSlot`/
  `TxSlot`) rather than the released `rx_done`/`tx_done` pair. Both
  requirements in `Cargo.toml` move to real versions when the upstream
  releases land, and nothing here can be published before then.

### Changed

- **Requires `rpi-hal` 0.3.0**, up from 0.1.0, which matters to an
  application more than a dependency bump usually does: this crate's
  published 0.1.0 asks for `rpi-hal` 0.1.0, so taking a newer HAL
  alongside it put *two* versions of the HAL in one graph. That fails as
  `time_driver::init` refusing a `Timer` — "expected
  `rpi_hal::timer::Timer`, found a different `rpi_hal::timer::Timer`" —
  which says nothing about the real cause. Pinning both halves of this
  release to 0.3.0 is what removes it.

  0.3.0 is also a floor rather than a preference: the `lan9514` module
  below is built on `Lan9514::split` and the LAN9514's `_async` methods,
  which no earlier release has.

  Nothing in this crate's own API changes, but the
  two `embassy-net` examples move to 0.2.0's owned USB `Channel`: they
  allocate a channel for the stack with `Dwc2Host::alloc_channel` rather
  than lending the whole controller, and `Lan9514Driver` grew a second
  lifetime for it. A consequence worth noting for anyone copying the
  pattern — the controller has to be widened to `'static` *before*
  `alloc_channel`, since a `Channel` borrows the controller it came from and
  the driver keeps its channel for the life of the program.

## [0.1.0] - 2026-08-12

### Added

- An `embassy-time` driver over the BCM System Timer, with the tick rate
  pinned to the hardware's fixed 1MHz rather than left to the
  application to choose.
- A thread-mode executor that behaves identically on AArch32 and
  AArch64, supplying its own `__pender` and idle loop rather than taking
  an upstream `platform-*` backend — `platform-cortex-ar` covers only
  AArch32, and `platform-spin` never idles the CPU.
- Examples exercising the driver and executor against real peripherals:
  blink, button, `Instant`/`Duration`, async UART echo, an
  `embassy-net` TCP echo server, and a `picoserve` HTTP server.

[0.1.0]: https://github.com/joeferner/rpi-hal-embassy/releases/tag/v0.1.0
