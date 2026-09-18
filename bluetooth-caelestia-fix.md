# Bluetooth Setup & Pairing Fix for Caelestia on CachyOS

## Overview

This document details the troubleshooting, diagnosis, and fix applied to enable Bluetooth device pairing and audio connectivity under **CachyOS** running the **Caelestia** desktop shell (Quickshell on Hyprland).

- **Hardware**: Intel Wi-Fi 7 / Bluetooth BE201 PCIe adapter (`btintel_pcie`)
- **System**: CachyOS (Linux kernel 7.2.x), PipeWire + WirePlumber audio stack
- **Desktop Environment**: Hyprland with Caelestia Shell (Quickshell)
- **Key Services**:
  - System service: `/etc/systemd/system/btintel-fix.service`
  - User service: `~/.config/systemd/user/bluetooth-agent.service`
  - Agent binary: `~/.local/bin/bluetooth-agent`

---

## 1. Hardware & Driver Context (Intel BE201 PCIe)

### The Boot Race Condition
The Intel BE201 PCIe Bluetooth module frequently encounters a timing/initialization race condition during early Linux boot, yielding a driver probe failure:
```text
btintel_pcie 0000:00:14.7: probe with driver btintel_pcie failed with error -62
```
Error `-62` (`-ETIME`) occurs when the firmware loading or PCIe initialization handshake times out before the device is ready.

### The System-Level Fix (`btintel-fix.service`)
To recover from this timeout, a systemd service is deployed at `/etc/systemd/system/btintel-fix.service`:

```ini
[Unit]
Description=Fix Intel BE201 Bluetooth PCIe race condition
After=network.target
Before=display-manager.service
ConditionPathExists=!/sys/class/bluetooth/hci0

[Service]
Type=oneshot
ExecStartPre=/usr/bin/sleep 1
ExecStart=/usr/bin/modprobe -r btintel_pcie
ExecStart=/usr/bin/modprobe btintel_pcie
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

**How it works**:
- `ConditionPathExists=!/sys/class/bluetooth/hci0`: If `hci0` was already initialized successfully, the service skips execution.
- If missing, it reloads `btintel_pcie` after a 1-second delay, allowing the kernel to re-probe the PCIe device and download the firmware files (`intel/ibt-0190-0291-pci.sfi` and `intel/ibt-0190-0291-pci.ddc`).
- Inspection confirmed that `btintel-fix.service` runs cleanly at boot and brings up `hci0` without errors.

---

## 2. The Problem: Unable to Connect in Caelestia UI

### Symptoms
- In Caelestia's UI, the Bluetooth toggle could turn Bluetooth on/off.
- However, clicking to connect or pair devices failed, and the saved devices list remained empty ("No saved devices").

### Root Cause Analysis

1. **Caelestia UI Structure**:
   - The main Bluetooth settings page (`/etc/xdg/quickshell/caelestia/modules/nexus/pages/BluetoothPage.qml`) filters devices strictly by bonded status:
     ```qml
     values: Bluetooth.devices.values.filter(d => d.bonded)
     ```
     If an apparatus is not already paired, it will not appear in the saved devices list.
   - The quick bar popout (`/etc/xdg/quickshell/caelestia/modules/bar/popouts/Bluetooth.qml`) calls `connected = !connected`, invoking `org.bluez.Device1.Connect()` directly on D-Bus. Calling `Connect()` on an unbonded device is rejected by BlueZ.
   - Pairing must be initiated through the sub-page `BluetoothPairing.qml` ("Pair new device"), which calls `modelData.pair()` (`org.bluez.Device1.Pair()`).

2. **Missing BlueZ Authentication Agent**:
   - BlueZ requires an active **Authentication Agent** (`org.bluez.Agent1`) registered with `org.bluez.AgentManager1` to handle link keys, PINs, and "Just Works" SSP (Secure Simple Pairing).
   - Neither Quickshell nor Caelestia registers a BlueZ agent.
   - In `~/.config/hypr/hyprland/execs.lua`, only a Polkit agent (`polkit-gnome`) and `mpris-proxy` were launched, with no Bluetooth agent present.
   - Without an agent:
     - BlueZ kept the controller state locked in `Pairable: no`.
     - Any call to `.pair()` failed immediately with `No agent available`.

---

## 3. Solution Implemented

### 1. Bluetooth Agent Script (`~/.local/bin/bluetooth-agent`)

A dedicated Python daemon runs BlueZ's built-in `bluetoothctl` with the `NoInputNoOutput` capability:

```python
#!/usr/bin/env python3
import signal
import subprocess
import sys

def main():
    proc = subprocess.Popen(
        ["/usr/bin/bluetoothctl", "--agent", "NoInputNoOutput"],
        stdin=subprocess.PIPE
    )

    def handle_signal(signum, frame):
        proc.terminate()
        try:
            proc.wait(timeout=3)
        except subprocess.TimeoutExpired:
            proc.kill()
        sys.exit(0)

    signal.signal(signal.SIGTERM, handle_signal)
    signal.signal(signal.SIGINT, handle_signal)

    exit_code = proc.wait()
    sys.exit(exit_code)

if __name__ == "__main__":
    main()
```

**Key Details**:
- **Capability `NoInputNoOutput`**: Informs BlueZ that the host acts as an auto-confirming agent for headless/desktop usage (ideal for audio headsets, mice, keyboards, and simple SSP pairing).
- **Persistent standard input**: `bluetoothctl` expects an open stdin stream; closing stdin causes it to terminate immediately. The wrapper maintains an open pipe.
- **Signal Handling**: Intercepts `SIGTERM` and `SIGINT` to gracefully terminate child processes when systemd stops or restarts the unit.

Make the script executable:
```bash
chmod +x ~/.local/bin/bluetooth-agent
```

### 2. User Systemd Service (`~/.config/systemd/user/bluetooth-agent.service`)

Created the systemd user service:

```ini
[Unit]
Description=Bluetooth Authentication Agent
After=bluetooth.target

[Service]
Type=simple
ExecStart=%h/.local/bin/bluetooth-agent
Restart=always
RestartSec=3

[Install]
WantedBy=default.target
```

Enabled and started the service:
```bash
systemctl --user daemon-reload
systemctl --user enable --now bluetooth-agent.service
```

---

## 4. Verification & Status

### Controller State
Verifying with `bluetoothctl show`:
```text
Controller 9C:67:D6:D2:68:A5 (public)
    Powered: yes
    Pairable: yes
    Discovering: yes
```
The controller is now permanently `Pairable: yes`.

### Device Pairing Test ("cerise" Headset)
- **MAC Address**: `68:F2:1F:20:5C:C9`
- **State**:
  - `Paired: yes`
  - `Bonded: yes`
  - `Trusted: yes`
  - `Connected: yes`
  - `Battery Percentage: 70%`
- **Audio Output**: WirePlumber automatically mapped the device as default:
  ```text
  Audio
   ├─ Devices:
   │     173. cerise                              [bluez5]
   ├─ Sinks:
   │  *  176. cerise                              [vol: 0.59]
  ```
- **Caelestia UI**: Appears in the "Connected devices" view with battery reporting and seamless connect/disconnect toggling.

---

## 5. Maintenance & Handy Commands

### Checking Agent Service
```bash
# Check service status
systemctl --user status bluetooth-agent.service

# View live logs
journalctl --user -u bluetooth-agent.service -f

# Restart service
systemctl --user restart bluetooth-agent.service
```

### Manual Bluetooth Commands
```bash
# Check adapter state
bluetoothctl show

# List paired devices
bluetoothctl devices Paired

# Connect / Disconnect manually
bluetoothctl connect <MAC_ADDRESS>
bluetoothctl disconnect <MAC_ADDRESS>

# Check audio sink status in PipeWire
wpctl status
```
