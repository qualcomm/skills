
---
name: configure-ethernet
description: Configure Ethernet features on the active Qualcomm Dragonwing device connected via qualcomm-ide MCP over SSH. Automatically executes commands on the device. Covers link speed, EEE, gPTP/TSN, MAC address, MTU, NIC settings, and DTS overlay. Platform: QCS6490 (RB3 Gen 2 Development Kit).

id: iot.qualcomm-linux.configure.ethernet
version: "1.0"
status: approved

tech_area: DTB Configuration
skill_category: configuration

confidence_level: advisory
output_type: procedure

depends_on:
related_skills:

applicable_products:
  - QCS6490
  - RB3 Gen 2 Development Kit
applicable_releases:
  - ">=QLI 2.0"
region: global

required_inputs:
  - active_device
  - action
  - interface
  - speed
  - autoneg
  - duplex
  - eee_state
  - ip_address
  - mtu_size
  - mac
  - ptp_role

document_references:
  - id: dragonwingdocs.qualcomm.com
    revision: unknown
    title: Qualcomm DragonWing Documentation
    section: Ethernet Configuration
    relationship: informational

known_gaps:
  - Does not cover static IP address persistence across reboots
  - Does not cover VLAN configuration
  - Does not cover network bonding, teaming, or bridging
  - Does not cover IPv6 configuration
  - Does not cover firewall or iptables rules
  - Does not cover Wi-Fi or cellular interfaces
  - Does not cover persistent MAC address configuration (runtime MAC changes reset on reboot)
  - Does not cover MAC EEPROM programming (QCS6490 has no EEPROM; MAC is stored in a persistent path on the device)
  - Does not cover QLI releases prior to 2.0
---

# Configure Ethernet Features

## Purpose & Scope

Covers runtime Ethernet configuration on the QCS6490 (RB3 Gen 2 Development Kit) device running Qualcomm Linux (QLI 2.0) via SSH through the qualcomm-ide MCP. The QCS6490 uses a QPS615 PCIe switch exposing two interfaces (`enP1p5s0f0` primary, `enP1p5s0f1` secondary). Features include link speed, EEE, temporary MAC address assignment, MTU adjustment, and NIC settings via `ethtool` and `ip` commands.

**In scope:**
- Runtime link speed configuration (10/100/1000/2500 Mbps)
- EEE enable/disable on the QEP8121 PHY
- Temporary MAC address assignment
- MTU adjustment
- NIC statistics and settings reporting via `ethtool`
- Interface status reporting (link state, interface list)

**Out of scope:**
- Static IP address persistence across reboots
- Persistent MAC address configuration (no EEPROM; MAC stored in device persistent path)
- VLAN, network bonding, teaming, or bridging configuration
- IPv6 configuration
- Firewall or iptables rules
- Wi-Fi or cellular interfaces
- gPTP/TSN (not supported on QCS6490)
- QLI releases prior to 2.0


## When to Use This Skill

**Invoke this skill when:**
- User wants to configure Ethernet on a QCS6490 (RB3 Gen 2) device running QLI 2.0
- User asks how to set link speed, enable EEE, change MAC address, or adjust MTU on QCS6490
- User wants to view NIC settings or interface status on QCS6490 / RB3 Gen 2

**Example queries:**
- "How do I set the Ethernet speed on my RB3 Gen 2?"
- "How do I enable EEE on QCS6490?"
- "How do I change the MAC address on my RB3 Gen 2?"

**Do not invoke this skill when:**
- The target device is not a QCS6490 / RB3 Gen 2 — use the platform-specific skill for IQ-9075, IQ-8275, or IQ-615 instead
- The user is asking about Wi-Fi or cellular configuration
- The user needs persistent MAC configuration (no EEPROM support — advise storing MAC in device persistent path)


## Required Inputs

Before applying this skill, the agent must have confirmed:

| Input | Description | Example |
|---|---|---|
| `active_device` | Currently selected device in qualcomm-ide MCP, must be active and SSH-reachable | QCS6490 / RB3 Gen 2 via SSH |
| `action` | Ethernet operation to perform | `link-speed`, `eee`, `mac-address`, `mtu`, `nic-settings`, `status`, `all` |
| `interface` | Ethernet interface name (optional — defaults to `enP1p5s0f0`) | `enP1p5s0f0`, `enP1p5s0f1` |
| `speed` | Link speed in Mbps (for `link-speed` action) | `2500` |
| `autoneg` | Auto-negotiation state (for `link-speed` action, default `on`) | `on` |
| `duplex` | Duplex mode (for `link-speed` action, default `full`) | `full` |
| `eee_state` | EEE enable/disable (for `eee` action on QEP8121 PHY) | `on` |
| `ip_address` | Static IP with prefix (for IP assignment) | `192.168.1.2/24` |
| `mtu_size` | MTU value in bytes (for `mtu` action, default `1500`) | `9000` |
| `mac` | MAC address in `XX:XX:XX:YY:YY:YY` format (for `mac-address` action) | `00:11:22:33:44:55` |
| `ptp_role` | PTP role (for `gptp` action; not supported on QCS6490) | `master` |

If any required input is missing, the agent should prompt the user before proceeding.


## Procedure / Decision Logic

The user provided these arguments: "$ARGUMENTS"

### Step 1 — Get active device and SSH credentials

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

### Step 2 — Map device to platform

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

### Step 3 — Parse requested action from arguments

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

### Step 4 — Execute platform-specific Ethernet commands over SSH

Use the `Bash` tool to run each command over SSH using the credentials from Step 1:
```bash
ssh -i $SSH_KEY -p $SSH_PORT -o StrictHostKeyChecking=no $SSH_USER@$SSH_HOST '<command>'
```

Show the command being run, execute it, and show the output. If a command fails, report the error and stop.

---

#### Platform: QCS6490 (RB3 Gen 2 Development Kit)

**Default interface:** `enP1p5s0f0` (primary), `enP1p5s0f1` (secondary)
**Supported speeds:** 10 / 100 / 1000 / 2500 Mbps
**Note:** Uses QPS615 PCIe switch — no EEPROM for MAC; MAC is stored in persistent path on device.

##### action=status
```bash
ssh ... 'ip link show && ethtool enP1p5s0f0'
```

##### action=link-speed
```bash
ssh ... 'ethtool -s <interface> autoneg <on|off> speed <10|100|1000|2500> duplex full'
# Example: ethtool -s enP1p5s0f0 autoneg on speed 2500 duplex full
```
Verify after:
```bash
ssh ... 'ethtool enP1p5s0f0'
```

##### action=eee (QEP8121 PHY only)
Check status:
```bash
ssh ... 'ethtool --show-eee <interface>'
```
Enable:
```bash
ssh ... 'ethtool --set-eee <interface> eee on'
```
Disable:
```bash
ssh ... 'ethtool --set-eee <interface> eee off'
```
Verify after:
```bash
ssh ... 'ethtool --show-eee <interface>'
```

##### action=mac-address (temporary — resets on reboot)
```bash
ssh ... 'ip link set <interface> down'
ssh ... 'ip link set dev <interface> address <mac>'
ssh ... 'ip link set dev <interface> up'
ssh ... 'ip -s addr show dev <interface>'
```

##### action=mtu
```bash
ssh ... 'ip link set dev <interface> down'
ssh ... 'ip link set dev <interface> mtu <mtu_size>'
ssh ... 'ip link set dev <interface> up'
ssh ... 'ip -s addr show dev <interface>'
```

##### action=nic-settings
```bash
ssh ... 'ethtool <interface>'
ssh ... 'ethtool -S <interface>'
```

---

### Step 5 — Report results

After executing each command:
1. Display the actual command run (with real SSH host/user substituted).
2. Show the full command output.
3. Confirm whether the configuration succeeded based on the output.
4. Warn about temporary settings that reset on reboot:
   - MAC address changes: all platforms
   - Static IP assignments: all platforms
5. For gPTP: note the daemon must stay running to maintain sync; suggest adding it to a systemd service for persistence.
6. For DTS overlay: confirm reboot is required; offer to reboot via `mcp__qualcomm-ide__reboot_device`.

### Step 6 — Offer next steps

Suggest follow-on actions:
- After link-speed: `ethtool <interface>` to confirm speed
- After mac-address: `ip link show <interface>` to confirm
- After mtu: `ip link show <interface>` to confirm
- After eee enable: `ethtool --show-eee <interface>` to verify
- After gptp: `grep -i ptp /var/log/syslog | tail -20` to monitor sync
- After dts-overlay: reboot then check `ip link show` for new interfaces
