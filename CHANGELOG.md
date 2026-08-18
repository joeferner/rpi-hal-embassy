# Changelog

Notable changes to `rpi-hal-embassy`, in the format of
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/). This crate
follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

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

### Changed

- The `embassy-net` examples take an owned `rpi_hal::usb::dwc2::Channel`
  where they took `&mut Dwc2Host`, following that crate's move to owned
  host-channel handles; `Lan9514Driver` now carries two lifetimes.

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
