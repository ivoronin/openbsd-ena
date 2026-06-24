# OpenBSD patches (separate from ena(4))

Fixes discovered while bringing OpenBSD/arm64 up natively on AWS Graviton.
They are independent of the ena(4) driver and are submittable to tech@ on
their own. Diffs are against -current (the OpenBSD master mirror).

## nvme-mqes-clamp.diff — nvme(4): clamp queue size to CAP.MQES

**Problem.** OpenBSD nvme(4) hardcodes 128-entry admin and I/O queues. The
AWS Nitro EBS NVMe controller on Graviton advertises `CAP.MQES = 32`, so
`ADD_IOCQ` is rejected with "Invalid Queue Size" (status type 1, code 0x02)
and the disk never attaches ("nvme0: unable to create io q") — no disk root,
no install.

**Fix.** In `nvme_attach` and `nvme_resume`, clamp the requested queue size
to `CAP.MQES`, and cap `sc_openings` (concurrent commands) at one below the
queue depth — `nvme_q_submit` has no full-SQ check, so allowing more
outstanding commands than the ring holds overruns it (corruption/panic under
load). General robustness fix; not AWS-specific.

**Tested.** On a real t4g (MQES=32): `sd0` attaches, fsck, `root on sd0a`,
multiuser boot, sustained I/O.

**Note.** `CAP.MQES` is 0-based per the NVMe spec (max queue size = MQES+1),
so clamping to `NVME_CAP_MQES(cap)` is conservative by one entry. Tested with
the conservative form; change to `NVME_CAP_MQES(cap) + 1` to use the full
controller maximum.

## spcr-console.diff — EC2/arm64 serial console via SPCR (com/acpi)

**Problem.** On AWS Graviton (and other ARM SBBR platforms) the serial
console UART is described only by the ACPI SPCR table at a fixed MMIO address
(0x90a0000 on EC2/arm64) and has **no _HID device** in the ACPI namespace.
OpenBSD attaches tty drivers from _HID matches, so no driver claims that UART,
nothing backs /dev/console, and init cannot open the console (the machine
boots but is silent / unusable without a working console device).

**Fix (3 files).**
- `dev/acpi/acpivar.h`: add `struct acpi_spcr *aaa_spcr` to the attach args.
- `dev/acpi/acpi.c`: `acpi_foundspcr()` finds the SPCR table and, when it
  describes a SystemMemory 16550/16450 with no matching _HID, synthesizes a
  `config_found()` attach for it (base address, size, GSIV from SPCR).
- `dev/acpi/com_acpi.c`: accept the SPCR-described UART, derive register
  width/shift and clock from the SPCR GAS, and mark it the console (dedups
  against any _HID device that already owns the same base via the existing
  console check).

**Tested.** On a real t4g: `com0 at acpi0 ... addr 0x90a0000`, `com0: console`,
multiuser boot with a working serial console.

**Note.** Uses existing `struct acpi_spcr` / `SPCR_SIG` / `GAS_*` definitions
from `dev/acpi/acpireg.h` (unchanged) and the existing
`com_acpi_intr_designware` / `com_acpi_is_console` helpers.
