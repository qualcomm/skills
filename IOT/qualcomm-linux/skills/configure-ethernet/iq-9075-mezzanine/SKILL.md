
---
name: configure-ethernet
description: Configure Ethernet features on the active Qualcomm Dragonwing device connected via qualcomm-ide MCP over SSH. Automatically executes commands on the device. Covers link speed, EEE, gPTP/TSN, MAC address, MTU, NIC settings, and DTS overlay. Platform: IQ-9075 with Mezzanine (IFP / GMSL Mezzanine Board).

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
  - QCS9075
  - IQ-9075 with Mezzanine (IFP / GMSL Mezzanine Board)
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
  - Does not cover gPTP/TSN on the mezzanine interface (gPTP is only supported on the on-board eth0; use IQ-9075 skill)
  - Does not cover the on-board eth0 interface when mezzanine is attached (use IQ-9075 skill instead)
  - DTS overlay written via efivar is not reversible without a second reboot; there is no automated rollback
  - Does not cover QLI releases prior to 2.0
---

# Configure Ethernet Features

## Purpose & Scope

Covers runtime Ethernet configuration on the QCS9075 (IQ-9075) device with an IFP or GMSL Mezzanine board attached, running Qualcomm Linux (QLI 2.0) via SSH through the qualcomm-ide MCP. The mezzanine exposes `enp5s0f0` via a QPS615 PCIe switch. An EFI-based DTS overlay must be applied (and the device rebooted) before the mezzanine Ethernet interfaces become available. Features include DTS overlay application, link speed adjustment (up to 10 Gbps), EEE enable/disable on the QEP8121 PHY, temporary MAC address assignment, MTU adjustment, and NIC settings.

**In scope:**
- EFI-based DTS overlay application to enable mezzanine Ethernet interfaces
- Runtime link speed configuration (100/1000/2500/5000/10000 Mbps) on `enp5s0f0`
- EEE enable/disable on the QEP8121 PHY (`enp5s0f0`)
- Temporary MAC address assignment
- MTU adjustment
- NIC statistics and settings reporting via `ethtool`
- Interface status reporting (link state, interface list)

**Out of scope:**
- Static IP address persistence across reboots
- Persistent MAC address configuration (runtime changes reset on reboot)
- VLAN, network bonding, teaming, or bridging configuration
- IPv6 configuration
- Firewall or iptables rules
- Wi-Fi or cellular interfaces
- gPTP/TSN on the mezzanine interface (only supported on on-board eth0; use IQ-9075 skill)
- On-board `eth0` interface (use the IQ-9075 without Mezzanine skill)
- DTS overlay rollback automation
- QLI releases prior to 2.0


## When to Use This Skill

**Invoke this skill when:**
- User wants to configure Ethernet on a QCS9075 (IQ-9075) device **with** a Mezzanine board attached, running QLI 2.0
- User needs to apply the EFI DTS overlay to enable mezzanine Ethernet interfaces
- User asks how to set link speed or enable EEE on the mezzanine interface (`enp5s0f0`)
- User wants to view NIC settings or interface status on IQ-9075 with Mezzanine

**Example queries:**
- "How do I enable Ethernet on my IQ-9075 with mezzanine?"
- "How do I apply the DTS overlay for the IQ-9075 mezzanine board?"
- "How do I enable EEE on enp5s0f0 on IQ-9075?"
- "How do I set 10G link speed on IQ-9075 mezzanine?"

**Do not invoke this skill when:**
- The target device is an IQ-9075 **without** a Mezzanine board — use the IQ-9075 skill instead
- The target device is not a QCS9075 / IQ-9075 — use the platform-specific skill for IQ-8275, QCS6490, or IQ-615 instead
- The user needs gPTP/TSN — use the IQ-9075 (without Mezzanine) skill instead


## Required Inputs

Before applying this skill, the agent must have confirmed:

| Input | Description | Example |
|---|---|---|
| `active_device` | Currently selected device in qualcomm-ide MCP, must be active and SSH-reachable | IQ-9075 with Mezzanine via SSH |
| `action` | Ethernet operation to perform | `link-speed`, `eee`, `dts-overlay`, `mtu`, `nic-settings`, `status`, `all` |
| `interface` | Ethernet interface name (optional — defaults to `enp5s0f0`) | `enp5s0f0` |
| `speed` | Link speed in Mbps (for `link-speed` action) | `10000` |
| `autoneg` | Auto-negotiation state (for `link-speed` action, default `on`) | `on` |
| `duplex` | Duplex mode (for `link-speed` action, default `full`) | `full` |
| `eee_state` | EEE enable/disable (for `eee` action on QEP8121 PHY) | `on` |
| `ip_address` | Static IP with prefix (for IP assignment) | `192.168.1.2/24` |
| `mtu_size` | MTU value in bytes (for `mtu` action, default `1500`) | `9000` |
| `mac` | MAC address in `XX:XX:XX:YY:YY:YY` format (for `mac-address` action; temporary) | `00:11:22:33:44:55` |
| `ptp_role` | PTP role (for `gptp` action; gPTP is only supported on on-board eth0 — use IQ-9075 skill) | `master` |

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

#### Platform: IQ-9075 with Mezzanine (IFP / GMSL Mezzanine Board)

**Default interface:** `enp5s0f0` (QPS615 PCIe switch)
**Supported speeds:** 100 / 1000 / 2500 / 5000 / 10000 Mbps
**Prerequisite:** EFI-based DTS overlay must be applied to enable Ethernet interfaces.

##### action=status
```bash
ssh ... 'ip link show && ethtool enp5s0f0 2>/dev/null || echo "Interface may require DTS overlay — run action=dts-overlay first"'
```

##### action=link-speed
```bash
ssh ... 'ethtool -s <interface> autoneg <on|off> speed <100|1000|2500|5000|10000> duplex full'
# Example: ethtool -s enp5s0f0 autoneg on speed 2500 duplex full
```
Verify:
```bash
ssh ... 'ethtool enp5s0f0'
```

##### action=eee (QEP8121 PHY only)
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
Supported EEE modes: 1000baseT/Full, 10000baseT/Full, 1000baseKX/Full, 10000baseKX4/Full, 10000baseKR/Full, 2500baseT/Full, 5000baseT/Full

##### action=dts-overlay (required before Ethernet works on mezzanine)
> EFI-based DTS overlay enables QPS615 Ethernet ports and also enables XO shutdown and S2R functionality. A reboot is required.

```bash
ssh ... 'echo -n "staging" > /var/data'
ssh ... 'efivar -n 882f8c2b-9646-435f-8de5-f208ff80c1bd-VendorDtbOverlays -w -f /var/data'
ssh ... 'efivar -n 882f8c2b-9646-435f-8de5-f208ff80c1bd-VendorDtbOverlays -p'
ssh ... 'sync'
```
Inform the user: "DTS overlay written. The device must be rebooted to activate Ethernet interfaces."
Ask: "Reboot the device now? (yes/no)"
If yes, call `mcp__qualcomm-ide__reboot_device` with rebootType=`NORMAL`.

##### action=nic-settings
```bash
ssh ... 'ethtool enp5s0f0'
ssh ... 'ethtool -S enp5s0f0'
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
