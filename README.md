# BEAU HYPERVISOR

```text
_____________________   _____   ____ ___  ________    _________
\______   \_   _____/  /  _  \ |    |   \ \_____  \  /   _____/
 |    |  _/|    __)_  /  /_\  \|    |   /  /   |   \ \_____  \
 |    |   \|        \/    |    \    |  /  /    |    \/        \
 |______  /_______  /\____|__  /______/   \_______  /_______  /
        \/        \/         \/                   \/        \/ (2026)
```

## Introduction

BEAU is a compact ARM64 hypervisor bring-up project for QEMU `virt` and rk356x
hardware. It runs at EL2 and focuses on a small, readable virtualization base
for mixed RTOS and Linux guests.

```text
┌──────────────────────────────┐
│ BEAU Hypervisor · ARM64 EL2  │
├──────────────┬───────────────┤
│ VM0 Zephyr   │ service VM    │
│ VM1 LK       │ prelaunch VM  │
│ VM2 Linux    │ prelaunch VM  │
└──────────────┴───────────────┘
```

## Source Base And License

Check [LICENSE](LICENSE)

---

Hustle Embedded OS.
