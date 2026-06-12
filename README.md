# CLAN HYPERVISOR

```text
 ______   __       ______   __   __     ______   ______
/\  ___\ /\ \     /\  __ \ /\ "-.\ \   /\  __ \ /\  ___\
\ \ \____\ \ \____\ \  __ \\ \ \-.  \  \ \ \/\ \\ \___  \
 \ \_____\\ \_____\\ \_\ \_\\ \_\\"\_\  \ \_____\\/\_____\
  \/_____/ \/_____/ \/_/\/_/ \/_/ \/_/   \/_____/ \/_____/
```

CLAN is an ARM64 hypervisor bring-up project for QEMU `virt`. It runs at EL2
and currently boots two static RTOS guests, Zephyr and LK, at EL1.

## System Overview

```text
┌──────────────────────────────┐
│ Host serial console          │
│ CLAN shell · vsh · debug     │
└───────────────┬──────────────┘
                │
┌───────────────▼──────────────────────────────────────────────┐
│ CLAN Hypervisor EL2                                          │
│                                                              │
│  ┌────────┐  ┌─────────┐  ┌────────┐  ┌──────┐  ┌────────┐   │
│  │ vCPU   │  │ stage-2 │  │ vGICv3 │  │ PSCI │  │ vPL011 │   │
│  └────────┘  └─────────┘  └────────┘  └──────┘  └────────┘   │
└───────────────┬───────────────────────────────┬──────────────┘
                │                               │
        ┌───────▼────────┐              ┌───────▼────────┐
        │ VM0 · Zephyr   │              │ VM1 · LK       │
        │ service VM     │              │ pre-launched VM│
        └────────────────┘              └────────────────┘
```

## Current Scope

| Area | Status |
| --- | --- |
| Platform | ARM64 QEMU `virt`, GICv3, EL2 virtualization |
| Guests | Zephyr service VM and LK pre-launched VM |
| CPU model | 8 pCPUs; VM0 and VM1 intentionally share pCPU3 |
| Memory | Static guest RAM, stage-2 identity mapping |
| Interrupts | Initial vGICv3 model, virtual timer injection |
| Console | CLAN shell, `vsh`, vPL011/vUART, async VM output |
| Debug | `vcpus`, `threads`, `sched`, `vmap`, `irqs`, `vdump`, `bench` |

## VM Layout

| VM | Guest | Role | Entry | RAM | pCPUs |
| --- | --- | --- | --- | --- | --- |
| VM0 | Zephyr | Service VM | `0x42000000` | `0x42000000-0x48000000` | 0, 2, 3, 4 |
| VM1 | LK | Pre-launched VM | `0x40100000` | `0x40000000-0x42000000` | 3, 5, 6, 7 |

```text
┌────────────────────────────── pCPU ownership ────────────────────────┐
│ pCPU    │  0  │  1  │  2  │     3 shared     │  4  │  5  │  6  │  7  │
├─────────┼─────┼─────┼─────┼──────────────────┼─────┼─────┼─────┼─────┤
│ guest   │ VM0 │  -  │ VM0 │     VM0 + VM1    │ VM0 │ VM1 │ VM1 │ VM1 │
│ role    │  Z  │  -  │  Z  │      Z + LK      │  Z  │ LK  │ LK  │ LK  │
└─────────┴─────┴─────┴─────┴──────────────────┴─────┴─────┴─────┴─────┘
```

## Key Features

```text
┌────────────────────────────── feature map ───────────────────────────────┐
│ ➊ CPU virtualization    EL1 guest entry/exit and vCPU-owned state        │
│ ➋ Memory virtualization Static guest RAM with stage-2 identity mappings  │
│ ➌ Interrupts            vGICv3 LRs, MMIO traps, virtual timer injection  │
│ ➍ Console               vPL011/vUART plus host-side vsh switching        │
│ ➎ Firmware ABI          PSCI CPU and system control virtualization       │
│ ➏ Observability         shell commands for VM, IRQ, map, and sched state │
└──────────────────────────────────────────────────────────────────────────┘
```

- Static QEMU board and VM configuration for fast RTOS bring-up.
- Raw Zephyr and LK images boot without external ACPI/FDT modules.
- Shared-pCPU scheduling saves and restores guest EL1, timer, and vGIC state.
- VM console output is buffered through async per-VM rings.

## Build And Run

```bash
./scripts/kick.py --toolchains /path/to/clan-arm64-none-elf/bin --build --dry-run
./scripts/kick.py --toolchains /path/to/clan-arm64-none-elf/bin --build
```

Run an existing image:

```bash
./scripts/kick.py
```

Inspect the generated QEMU command:

```bash
./scripts/kick.py --dry-run
```

Default QEMU shape:

```text
qemu-system-aarch64
  -machine virt,virtualization=on,gic-version=3
  -cpu cortex-a57
  -smp 8
  -m 1024M
  -nographic
  -serial mon:stdio
  -kernel build/acrn.out
```

## Runtime Checks

| Command | Purpose |
| --- | --- |
| `vcpus` | Show VM/vCPU placement and state |
| `sched` | Show scheduler ticks and context switches |
| `vmap` | Show host and guest memory mappings |
| `irqs` | Show interrupt names, state, and per-pCPU counts |
| `vdump [vm] [vcpu]` | Dump VM/vCPU register state |
| `vsh 0` | Enter Zephyr console; expect `zero ~>` |
| `vsh 1` | Enter LK console; expect `beau ~>` |
| Ctrl-D | Return from a VM console to the CLAN shell |

## Verified Behavior

```text
┌────────────────────────────── verified ──────────────────────────────┐
│ ✅ QEMU boots with -smp 8 and reaches the CLAN shell                 │
│ ✅ Zephyr and LK autostart without external boot modules             │
│ ✅ VM0 and VM1 run concurrently and share pCPU3 through sched_iorr   │
│ ✅ VM RAM is mapped IPA/GPA equal to PA/HPA                          │
│ ✅ vsh 0 and vsh 1 reach the expected guest shells                   │
│ ✅ reboot resets QEMU and restarts CLAN                              │
└──────────────────────────────────────────────────────────────────────┘
```

## Development Plan

```text
┌────┬───────────────────────────┬────────────────────────────────────────────┐
│ P0 │ Linux loader path         │ Boot richer guests without .incbin images  │
│ P0 │ vGICv3 coverage           │ Add routing, active-state, SGI/PPI paths   │
│ P1 │ Boot regression           │ Automate build, QEMU, vsh, vmap, irqs      │
│ P1 │ FDT-derived platform data │ Reduce static QEMU board assumptions       │
│ P1 │ Console regression        │ Test vsh, Ctrl-D, async output, overflow   │
│ P2 │ Better observability      │ Expand VM and interrupt debug views        │
└────┴───────────────────────────┴────────────────────────────────────────────┘
```

## Design Direction

```text
┌──────────────────────┐        ┌──────────────────────┐
│ Current              │  ───▶  │ Next                 │
├──────────────────────┤        ├──────────────────────┤
│ Static RTOS images   │        │ Loader/module images │
│ Static QEMU layout   │        │ FDT-derived layout   │
│ Bring-up vGICv3      │        │ Broader GICv3 model  │
│ Manual boot checks   │        │ Automated regression │
└──────────────────────┘        └──────────────────────┘
```

CLAN is still a bring-up target. The near-term goal is a small, testable ARM64
virtualization base that can grow from static RTOS guests toward richer guest
boot flows.

---

Hustle Embedded OS.
