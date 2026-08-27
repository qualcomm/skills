
---
name: configure-ethernet
description: Configure Ethernet features on the active Qualcomm Dragonwing device connected via qualcomm-ide MCP over SSH. Automatically executes commands on the device. Covers link speed, EEE, gPTP/TSN, MAC address, MTU, NIC settings, and DTS overlay. Platform: IQ-615 (QCS615).

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
  - QCS615
  - IQ-615
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
  - Does not cover persistent MAC address configuration (MAC resets to a random address on every reboot)
  - Does not cover EEE (not supported on QCS615)
  - Does not cover gPTP/TSN (not supported on QCS615)
  - Does not cover QLI releases prior to 2.0
---

# Configure Ethernet Features

## Purpose & Scope

Covers runtime Ethernet configuration on the QCS615 (IQ-615) device running Qualcomm Linux (QLI 2.0) via SSH through the qualcomm-ide MCP. The IQ-615 exposes a single `eth0` interface. A notable platform characteristic is that the MAC address resets to a random value on every reboot. Features include link speed adjustment, temporary MAC address assignment, MTU adjustment, and NIC settings via `ethtool` and `ip` commands.

**In scope:**
- Runtime link speed configuration
- Temporary MAC address assignment
- MTU adjustment
- NIC statistics and settings reporting via `ethtool`
- Interface status reporting (link state, interface list)

**Out of scope:**
- Persistent MAC address configuration (resets to random on reboot)
- Static IP address persistence across reboots
- VLAN, network bonding, teaming, or bridging configuration
- IPv6 configuration
- Firewall or iptables rules
- Wi-Fi or cellular interfaces
- EEE (not supported on QCS615)
- gPTP/TSN (not supported on QCS615)
- QLI releases prior to 2.0


## When to Use This Skill

**Invoke this skill when:**
- User wants to configure Ethernet on a QCS615 (IQ-615) device running QLI 2.0
- User asks how to set link speed, change MAC address, or adjust MTU on IQ-615
- User wants to view NIC settings or interface status on IQ-615 / QCS615

**Example queries:**
- "How do I set the Ethernet speed on my IQ-615?"
- "How do I change the MAC address on QCS615?"
- "How do I check Ethernet NIC statistics on my IQ-615?"

**Do not invoke this skill when:**
- The target device is not a QCS615 / IQ-615 — use the platform-specific skill for IQ-9075, IQ-8275, or QCS6490 instead
- The user needs persistent MAC configuration (not possible on IQ-615; MAC randomises on reboot)
- The user is asking about EEE or gPTP (not supported on QCS615)


## Required Inputs

Before applying this skill, the agent must have confirmed:

| Input | Description | Example |
|---|---|---|
| `active_device` | Currently selected device in qualcomm-ide MCP, must be active and SSH-reachable | IQ-615 / QCS615 via SSH |
| `action` | Ethernet operation to perform | `link-speed`, `mac-address`, `mtu`, `nic-settings`, `status`, `all` |
| `interface` | Ethernet interface name (optional — defaults to `eth0`) | `eth0` |
| `speed` | Link speed in Mbps (for `link-speed` action) | `1000` |
| `autoneg` | Auto-negotiation state (for `link-speed` action, default `on`) | `on` |
| `duplex` | Duplex mode (for `link-speed` action, default `full`) | `full` |
| `eee_state` | EEE enable/disable (not supported on QCS615) | `on` |
| `ip_address` | Static IP with prefix (for IP assignment) | `192.168.1.2/24` |
| `mtu_size` | MTU value in bytes (for `mtu` action, default `1500`) | `9000` |
| `mac` | MAC address in `XX:XX:XX:YY:YY:YY` format (for `mac-address` action; resets on reboot) | `00:11:22:33:44:55` |
| `ptp_role` | PTP role (for `gptp` action; not supported on QCS615) | `master` |

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

#### Platform: IQ-615 (QCS615)

**Default interface:** `eth0`
**Note:** MAC address resets to a random address on every reboot.

##### action=status
```bash
ssh ... 'ip link show && ethtool eth0'
```

##### action=link-speed
Check supported modes first:
```bash
ssh ... 'ethtool eth0 | grep -A 20 "Supported link modes"'
```
Set speed:
```bash
ssh ... 'ethtool -s <interface> autoneg <on|off> speed <speed> duplex full'
```

##### action=mac-address (temporary — resets to random on reboot)
```bash
ssh ... 'ip link set dev eth0 address <mac>'
ssh ... 'ip link show eth0'
```

##### action=mtu
```bash
ssh ... 'ip link set dev eth0 down && ip link set dev eth0 mtu <mtu_size> && ip link set dev eth0 up'
ssh ... 'ip link show eth0'
```

##### action=nic-settings
```bash
ssh ... 'ethtool eth0'
ssh ... 'ethtool -S eth0'
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
