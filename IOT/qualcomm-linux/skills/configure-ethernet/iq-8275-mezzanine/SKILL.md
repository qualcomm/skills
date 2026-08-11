
---
name: configure-ethernet
description: Configure Ethernet features on the active Qualcomm Dragonwing device connected via qualcomm-ide MCP over SSH. Automatically executes commands on the device. Covers link speed, EEE, gPTP/TSN, MAC address, MTU, NIC settings, and DTS overlay. Platform: IQ-8275 with Mezzanine (IFP / GMSL Mezzanine Board).
---

# Configure Ethernet Features

The user provided these arguments: "$ARGUMENTS"

## Step 1 — Get active device and SSH credentials

Call `mcp__qualcomm-ide__get_active_device` to retrieve the currently selected device.

If no device is active (empty or null response), stop and tell the user:
> "No active device found. Please connect and select a device first using the Qualcomm IDE device panel."

Extract the SSH connection details from the device response:
- `SSH_HOST` = `sshConfig.host`
- `SSH_PORT` = `sshConfig.port`
- `SSH_USER` = `sshConfig.username`
- `SSH_KEY`  = `sshConfig.keyPath`

Call `mcp__qualcomm-ide__validate_ssh_connection` with the device serial number to confirm SSH is reachable.
If validation fails, stop and tell the user to check the SSH connection.

Define a helper pattern for all subsequent remote commands:
```
ssh -i $SSH_KEY -p $SSH_PORT -o StrictHostKeyChecking=no $SSH_USER@$SSH_HOST '<command>'
```

## Step 2 — Map device to platform

Use the active device's chipset or display name to determine which platform configuration applies:

| Chipset / Display Name | Platform |
|---|---|
| QCS6490 / RB3 Gen 2 | **QCS6490** |
| QCS9075 / IQ-9075 (no mezzanine) | **IQ-9075** |
| QCS9075 / IQ-9075 + Mezzanine | **IQ-9075 with Mezzanine** |
| QCS8275 / IQ-8275 (no mezzanine) | **IQ-8275** |
| QCS8275 / IQ-8275 + Mezzanine | **IQ-8275 with Mezzanine** |
| IQ-615 / QCS615 | **IQ-615** |

If the mezzanine variant cannot be determined from device info alone, ask the user:
> "Is a Mezzanine board (IFP or GMSL) attached to your device? (yes/no)"

## Step 3 — Parse requested action from arguments

Parse `$ARGUMENTS` to extract:
- `action`: one of `link-speed`, `eee`, `gptp`, `mac-address`, `mtu`, `nic-settings`, `dts-overlay`, `status`, or `all`
- `interface`: Ethernet interface name (optional — use platform default if not provided)
- `speed`: numeric speed value in Mbps (for link-speed action)
- `autoneg`: `on` or `off` (for link-speed, default `on`)
- `duplex`: `full` or `half` (default `full`)
- `eee_state`: `on` or `off` (for eee action)
- `ip_address`: IP/prefix e.g. `192.168.1.2/24` (for static IP assignment)
- `mtu_size`: MTU value (default `1500`)
- `mac`: MAC address in `XX:XX:XX:YY:YY:YY` format
- `ptp_role`: `master` or `slave` (for gptp action)

If `action` is not provided, run `status` first (show interface list and link state), then ask the user which feature to configure.

## Step 4 — Execute platform-specific Ethernet commands over SSH

Use the `Bash` tool to run each command over SSH using the credentials from Step 1:
```bash
ssh -i $SSH_KEY -p $SSH_PORT -o StrictHostKeyChecking=no $SSH_USER@$SSH_HOST '<command>'
```

Show the command being run, execute it, and show the output. If a command fails, report the error and stop.

---

### Platform: IQ-8275 (QCS8275)

**Default interface:** `end0`
**Supported speeds:** 10 / 100 / 1000 / 2500 Mbps
**Feature set:** Basic Ethernet — interface enumeration and data path.

#### action=status
```bash
ssh ... 'ip link show && ethtool end0'
```

#### action=link-speed
```bash
ssh ... 'ethtool -s <interface> autoneg <on|off> speed <10|100|1000|2500> duplex full'
# Example: ethtool -s end0 autoneg on speed 2500 duplex full
```
Verify:
```bash
ssh ... 'ethtool end0'
```

#### action=mac-address (temporary — resets on reboot)
```bash
ssh ... 'ip link set dev end0 address <mac>'
ssh ... 'ip link show end0'
```

#### action=mtu
```bash
ssh ... 'ip link set dev end0 down && ip link set dev end0 mtu <mtu_size> && ip link set dev end0 up'
ssh ... 'ip link show end0'
```

#### action=nic-settings
```bash
ssh ... 'ethtool end0'
ssh ... 'ethtool -S end0'
```

---

### Platform: IQ-8275 with Mezzanine (IFP / GMSL Mezzanine Board)

**Default interface:** `enp5s0f0` (QPS615 PCIe switch)
**Supported speeds:** 100 / 1000 / 2500 Mbps
**Special feature:** EEE (Energy Efficient Ethernet) on QEP8121 PHY

#### action=status
```bash
ssh ... 'ip link show && ethtool enp5s0f0 2>/dev/null || echo "Interface not found — check mezzanine connection"'
```

#### action=link-speed
```bash
ssh ... 'ethtool -s <interface> autoneg <on|off> speed <100|1000|2500> duplex full'
# Example: ethtool -s enp5s0f0 autoneg on speed 2500 duplex full
```
Verify:
```bash
ssh ... 'ethtool enp5s0f0'
```

#### action=eee (QEP8121 PHY only)
Check status:
```bash
ssh ... 'ethtool --show-eee enp5s0f0'
```
Enable:
```bash
ssh ... 'ethtool --set-eee enp5s0f0 eee on'
```
Disable:
```bash
ssh ... 'ethtool --set-eee enp5s0f0 eee off'
```

#### action=nic-settings
```bash
ssh ... 'ethtool enp5s0f0'
ssh ... 'ethtool -S enp5s0f0'
```

---

## Step 5 — Report results

After executing each command:
1. Display the actual command run (with real SSH host/user substituted).
2. Show the full command output.
3. Confirm whether the configuration succeeded based on the output.
4. Warn about temporary settings that reset on reboot:
   - MAC address changes: all platforms
   - Static IP assignments: all platforms
5. For gPTP: note the daemon must stay running to maintain sync; suggest adding it to a systemd service for persistence.
6. For DTS overlay: confirm reboot is required; offer to reboot via `mcp__qualcomm-ide__reboot_device`.

## Step 6 — Offer next steps

Suggest follow-on actions:
- After link-speed: `ethtool <interface>` to confirm speed
- After mac-address: `ip link show <interface>` to confirm
- After mtu: `ip link show <interface>` to confirm
- After eee enable: `ethtool --show-eee <interface>` to verify
- After gptp: `grep -i ptp /var/log/syslog | tail -20` to monitor sync
- After dts-overlay: reboot then check `ip link show` for new interfaces
