
---
name: configure-fan-controller
description: Configure the AMC6821 PWM fan controller on Qualcomm Linux (QLI 2.0) — source-level (DTS patch, kernel config, Yocto recipe) and runtime (sysfs over SSH). Platform: IQ-9075 EVK (QCS9075).

id: iot.qualcomm-linux.configure.fan
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
  - IQ-9075 EVK
applicable_releases:
  - ">=QLI 2.0"
region: global

required_inputs:
  - action
  - source_dir
  - active_device
  - pwm_value
  - point1_temp
  - point2_temp
  - point3_temp
  - hwmon_num
  - platform

document_references:
  - id: dragonwingdocs.qualcomm.com
    revision: unknown
    title: Qualcomm DragonWing Documentation
    section: Fan Controller Configuration
    relationship: informational

known_gaps:
  - Does not cover platforms other than IQ-9075 EVK (QCS9075)
  - Does not cover fan controllers other than the TI AMC6821
  - Does not cover fan controllers connected via SPI or GPIO (only I2C-based)
  - Persistent runtime configuration via systemd is described but not fully automated end-to-end
  - Does not cover thermal zone tuning beyond the AMC6821 cooling map in the device tree
  - Does not cover building the kernel outside the Yocto/kas-container workflow
  - Does not cover QLI releases prior to 2.0
  - Does not cover reverting DTS or kernel config changes once applied (manual rollback required)
---

# Configure Temperature Regulation with Fan Controller

## Purpose & Scope

Covers configuration of the TI AMC6821 PWM fan controller on the IQ-9075 EVK (QCS9075) platform running Qualcomm Linux (QLI 2.0). Supports both **source-level** configuration (DTS patch, kernel config fragment, Yocto recipe edit) and **runtime** configuration (sysfs nodes on a running device via SSH through the qualcomm-ide MCP).

The AMC6821 is connected to the IQ-9075 EVK via I2C19 (QUP2 SE5) using GPIO99 (SDA) and GPIO100 (SCL), at I2C address `0x18`.

**In scope:**
- Creating and registering a DTS patch to enable I2C19 and the AMC6821 node in the QLI 2.0 Yocto tree
- Enabling `CONFIG_SENSORS_AMC6821` and `CONFIG_HWMON` in the kernel config fragment
- Runtime fan mode switching: automatic (temperature-controlled) and manual (fixed PWM)
- Runtime temperature threshold configuration via sysfs
- Fan controller status reporting (mode, PWM value, RPM, temperature)
- Optional systemd service creation for runtime persistence across reboots

**Out of scope:**
- Platforms other than IQ-9075 EVK (QCS9075)
- Fan controllers other than the TI AMC6821
- Fan controllers connected via SPI or GPIO
- Thermal zone tuning beyond the AMC6821 cooling map
- Building the kernel outside the Yocto/kas-container workflow
- Reverting DTS or kernel config changes (manual rollback required)
- QLI releases prior to 2.0


## When to Use This Skill

**Invoke this skill when:**
- User wants to enable or configure the AMC6821 fan controller on IQ-9075 EVK running QLI 2.0
- User asks how to add fan control support to the QLI 2.0 kernel build
- User wants to set fan speed, change fan mode, or adjust temperature thresholds at runtime
- User needs to make fan controller settings persist across reboots

**Example queries:**
- "How do I enable the fan controller on IQ-9075 EVK?"
- "How do I set the fan to automatic mode on QCS9075?"
- "How do I configure the AMC6821 temperature thresholds?"
- "How do I add AMC6821 support to the QLI 2.0 Yocto build?"

**Do not invoke this skill when:**
- The target platform is not an IQ-9075 EVK (QCS9075)
- The fan controller is not the TI AMC6821
- The user is asking about thermal management features other than the AMC6821 cooling map


## Required Inputs

Before applying this skill, the agent must have confirmed:

| Input | Description | Example |
|---|---|---|
| `action` | Operation to perform; one of `status`, `enable-dts`, `kernel-config`, `auto-mode`, `manual-mode`, `set-thresholds`, `status-runtime`, `all-source`, `all-runtime` | `auto-mode` |
| `source_dir` | Absolute path to the root of the QLI 2.0 checkout (required for all source-level actions; must contain `meta-qcom/`) | `/home/user/my-qli20-checkout` |
| `active_device` | Currently selected device in qualcomm-ide MCP (required for all runtime actions) | IQ-9075 EVK via SSH |
| `pwm_value` | Fan PWM value 0–255 (for `manual-mode` action) | `128` |
| `point1_temp` | Fan-off threshold in °C (for `set-thresholds` action, default `0`) | `0` |
| `point2_temp` | Minimum-speed threshold in °C (for `set-thresholds` action, default `48`) | `48` |
| `point3_temp` | Full-speed threshold in °C (for `set-thresholds` action, default `58`) | `58` |
| `hwmon_num` | hwmon index number (optional — auto-detected from `amc6821` name, fallback `0`) | `0` |
| `platform` | Target platform (optional — default `iq-9075`) | `iq-9075` |

If any required input is missing, the agent should prompt the user before proceeding.


## Procedure / Decision Logic

The user provided these arguments: "$ARGUMENTS"

---

### Step 1 — Parse requested action and resolve source directory

Parse `$ARGUMENTS` to extract:
- `action`: one of `status`, `enable-dts`, `kernel-config`, `auto-mode`, `manual-mode`,
  `set-thresholds`, `status-runtime`, `all-source`, `all-runtime`
- `source_dir`: **required for all source-level actions** — the absolute or `~`-expanded
  path to the root of the user's QLI 2.0 checkout (the directory that contains
  `meta-qcom/`, `meta-qcom-releases/`, `kas-container`, etc.)
- `pwm_value`: 0–255 (for `manual-mode`)
- `point1_temp`, `point2_temp`, `point3_temp`: threshold temps in °C (for `set-thresholds`)
- `hwmon_num`: hwmon index number (default: auto-detect from `amc6821` name, fallback `0`)
- `platform`: target platform (default: `iq-9075`)

**If `source_dir` is not provided and the action requires source edits**, ask the user:
> "Please provide the path to your QLI 2.0 source directory
> (e.g. `/home/user/my-qli20-checkout` — the directory that contains `meta-qcom/`):"

Once `source_dir` is known, validate it before proceeding:
```bash
test -d "$source_dir/meta-qcom" || echo "ERROR: meta-qcom/ not found in $source_dir"
test -f "$source_dir/kas-container"  || echo "WARNING: kas-container not found in $source_dir"
```
If `meta-qcom/` is missing, stop and tell the user the path does not look like a
QLI 2.0 source tree.

All paths in the steps below use `$source_dir` as the root. The internal structure
relative to `$source_dir` is fixed across all QLI 2.0 checkouts.

If `action` is not provided, run `status` first (describe source files and current state),
then ask the user which action to perform.

---

### Step 2 — Source-level actions (QLI 2.0 tree)

These actions modify source files in the QLI 2.0 Yocto tree. Use the `Bash` and `Edit`/`Write` tools.

#### action=status (source)

Report what currently exists in the tree:

```bash
# Check DTS for amc6821 / i2c19 / qup2_se5 nodes
grep -rn "amc6821\|i2c19\|qup2_se5\|gpio99\|gpio100\|fan" \
    <source_dir>/meta-qcom/recipes-kernel/linux/ 2>/dev/null

# Check kernel config fragments
grep -n "HWMON\|AMC6821\|SENSORS_AMC6821\|I2C" \
    <source_dir>/meta-qcom/recipes-kernel/linux/linux-qcom-6.18/configs/bsp-additions.cfg

# Check machine conf for DT overlays
grep -n "lemans\|fan\|i2c" \
    <source_dir>/meta-qcom/conf/machine/iq-9075-evk.conf
```

Display the results and summarise what is already present vs what needs to be added.

---

#### action=enable-dts

Enable the AMC6821 fan controller in the IQ-9075 EVK device tree.

**Background:**
- DTS file: `arch/arm64/boot/dts/qcom/qcs9075-iq-9075-evk.dts` (in kernel source)
- The I2C19 bus node (`serial@994000`) and its pinctrl (`qup2_se5`) must be enabled.
- The AMC6821 is an I2C slave at address `0x18` on I2C bus 19.

Because the kernel source is fetched during build, changes are delivered as a **patch file**
placed in the Yocto layer. Follow these steps:

**Step 2a — Create the DTS patch file**

Create the patch at:
```
<source_dir>/meta-qcom/recipes-kernel/linux/linux-qcom-6.18/
    0002-arm64-dts-qcom-qcs9075-iq-9075-evk-Enable-AMC6821-fa.patch
```

Patch content:
```diff
From: Developer <dev@example.com>
Date: $(date -R)
Subject: [PATCH] arm64: dts: qcom: qcs9075-iq-9075-evk: Enable AMC6821 fan controller

Enable I2C19 (QUP2 SE5) bus and add AMC6821 PWM fan controller node
on the IQ-9075 EVK board.

GPIO99 = I2C19 SDA (qup2_se5 function)
GPIO100 = I2C19 SCL (qup2_se5 function)
AMC6821 I2C address: 0x18

---
 .../boot/dts/qcom/qcs9075-iq-9075-evk.dts    | 34 +++++++++++++++++++
 1 file changed, 34 insertions(+)

diff --git a/arch/arm64/boot/dts/qcom/qcs9075-iq-9075-evk.dts \
    b/arch/arm64/boot/dts/qcom/qcs9075-iq-9075-evk.dts
--- a/arch/arm64/boot/dts/qcom/qcs9075-iq-9075-evk.dts
+++ b/arch/arm64/boot/dts/qcom/qcs9075-iq-9075-evk.dts
@@ -1,6 +1,40 @@
 // SPDX-License-Identifier: BSD-3-Clause
 /*
- * Copyright (c) 2024 Qualcomm Innovation Center, Inc. All rights reserved.
+ * Copyright (c) 2024-2025 Qualcomm Innovation Center, Inc. All rights reserved.
  */

 #include "qcs9075.dtsi"
+
+&qup_i2c19_default {
+	pins = "gpio99", "gpio100";
+	function = "qup2_se5";
+	drive-strength = <2>;
+	bias-pull-up;
+};
+
+/* Enable I2C19 (QUP2 SE5) for AMC6821 fan controller */
+&i2c19 {
+	status = "okay";
+	pinctrl-names = "default";
+	pinctrl-0 = <&qup_i2c19_default>;
+
+	fan_controller: amc6821@18 {
+		compatible = "ti,amc6821";
+		reg = <0x18>;
+		#cooling-cells = <2>;
+	};
+};
+
+/* Link fan controller as a cooling device in the CPU thermal zone */
+&cpu_thermal {
+	cooling-maps {
+		map0 {
+			trip = <&cpu_alert0>;
+			cooling-device = <&fan_controller 0 255>;
+		};
+	};
+};
```

> **Note:** The exact thermal zone label (`cpu_thermal`, `cpu_alert0`) depends on the
> platform's `lemans.dtsi`. Check `arch/arm64/boot/dts/qcom/lemans.dtsi` in the kernel
> source for the actual zone and trip-point names. Adjust the patch accordingly.

**Step 2b — Register the patch in linux-qcom_6.18.bb**

Add the patch to `SRC_URI` in:
```
<source_dir>/meta-qcom/recipes-kernel/linux/linux-qcom_6.18.bb
```

Append inside the first `SRC_URI` block:
```
    file://linux-qcom-6.18/0002-arm64-dts-qcom-qcs9075-iq-9075-evk-Enable-AMC6821-fa.patch \
```

Display both the created patch file path and the modified SRC_URI line. Confirm to the user
that these changes will be applied during the next `kas-container build`.

---

#### action=kernel-config

Enable the AMC6821 hwmon driver in the kernel config fragment.

**File to edit:**
```
<source_dir>/meta-qcom/recipes-kernel/linux/linux-qcom-6.18/configs/bsp-additions.cfg
```

**Lines to append:**
```cfg
# AMC6821 PWM fan controller (hwmon)
CONFIG_HWMON=y
CONFIG_SENSORS_AMC6821=y
CONFIG_I2C=y
```

Steps:
1. Read the file: check if `CONFIG_HWMON`, `CONFIG_SENSORS_AMC6821` are already present.
2. If missing, append the block above to the end of the file.
3. Display the diff of what was added.

> `CONFIG_THERMAL_HWMON=y` is already present in
> `bsp/qcom-armv8a/qcom.cfg` — no need to add it again.

---

#### action=all-source

Run in sequence: `status` → `kernel-config` → `enable-dts`.

Display a summary of all files modified.

---

### Step 3 — Runtime actions (on a running device)

These actions interact with the live device over SSH. They require an active SSH connection.

**Get SSH credentials:**
- Call `mcp__qualcomm-ide__get_active_device` to retrieve the selected device.
- Extract `SSH_HOST`, `SSH_PORT`, `SSH_USER`, `SSH_KEY` from `sshConfig`.
- Call `mcp__qualcomm-ide__validate_ssh_connection` to confirm SSH is reachable.
- If no device is active or SSH fails, inform the user and stop.

Define helper:
```bash
SSH="ssh -i $SSH_KEY -p $SSH_PORT -o StrictHostKeyChecking=no $SSH_USER@$SSH_HOST"
```

**Auto-detect hwmon number:**
```bash
$SSH 'for d in /sys/class/hwmon/hwmon*; do
    name=$(cat "$d/name" 2>/dev/null)
    [ "$name" = "amc6821" ] && echo "${d##*/}" && break
done'
```
Store result as `HWMON`. If empty, warn user and fall back to `hwmon0`.

---

#### action=status-runtime

Show full fan controller state on the running device:

```bash
$SSH "
HWMON=\$(for d in /sys/class/hwmon/hwmon*; do
    name=\$(cat \"\$d/name\" 2>/dev/null)
    [ \"\$name\" = \"amc6821\" ] && echo \"\${d##*/}\" && break
done)
HWMON=\${HWMON:-hwmon0}
BASE=/sys/class/hwmon/\$HWMON

echo '=== AMC6821 Fan Controller Status ==='
echo \"hwmon device : \$HWMON  (\$(cat \$BASE/name 2>/dev/null))\"
echo \"Mode         : \$(cat \$BASE/pwm1_enable 2>/dev/null)  (1=manual 2=auto)\"
echo \"PWM value    : \$(cat \$BASE/pwm1 2>/dev/null)  (0-255)\"
echo \"Fan RPM      : \$(cat \$BASE/fan1_input 2>/dev/null)\"
echo \"Remote temp  : \$(cat \$BASE/temp2_input 2>/dev/null) milli°C\"
echo ''
echo '=== Auto-mode Thresholds ==='
echo \"point1 (fan off)      : \$(cat \$BASE/temp2_auto_point1_temp 2>/dev/null) m°C\"
echo \"point2 (min speed)    : \$(cat \$BASE/temp2_auto_point2_temp 2>/dev/null) m°C\"
echo \"point3 (max speed)    : \$(cat \$BASE/temp2_auto_point3_temp 2>/dev/null) m°C\"
"
```

---

#### action=auto-mode

Switch to automatic temperature-controlled fan speed (default/recommended):

```bash
$SSH "echo 2 > /sys/class/hwmon/$HWMON/pwm1_enable"
$SSH "cat /sys/class/hwmon/$HWMON/pwm1_enable"
```

Confirm output is `2`. Note: this setting resets on reboot.

---

#### action=manual-mode

Switch to manual mode and set a specific fan speed (`pwm_value` = 0–255):

```bash
$SSH "echo 1 > /sys/class/hwmon/$HWMON/pwm1_enable"
$SSH "echo <pwm_value> > /sys/class/hwmon/$HWMON/pwm1"
$SSH "cat /sys/class/hwmon/$HWMON/pwm1_enable"
$SSH "cat /sys/class/hwmon/$HWMON/pwm1"
$SSH "cat /sys/class/hwmon/$HWMON/fan1_input"
```

If `pwm_value` is not provided, prompt the user:
> "Enter PWM value (0 = fan off, 128 = 50%, 255 = full speed):"

Warn: settings reset on reboot. For persistence, create a systemd unit (see Step 4).

---

#### action=set-thresholds

Set custom automatic-mode temperature thresholds (values in °C, converted to milli°C):

| Argument | Default °C | DTS equivalent |
|---|---|---|
| `point1_temp` | 0 | Fan off below this temperature |
| `point2_temp` | 48 | Fan runs at minimal speed above this |
| `point3_temp` | 58 | Fan runs at full speed above this |

```bash
P1=$(( <point1_temp> * 1000 ))
P2=$(( <point2_temp> * 1000 ))
P3=$(( <point3_temp> * 1000 ))

$SSH "echo $P1 > /sys/class/hwmon/$HWMON/temp2_auto_point1_temp"
$SSH "echo $P2 > /sys/class/hwmon/$HWMON/temp2_auto_point2_temp"
$SSH "echo $P3 > /sys/class/hwmon/$HWMON/temp2_auto_point3_temp"

# Verify
$SSH "cat /sys/class/hwmon/$HWMON/temp2_auto_point1_temp"
$SSH "cat /sys/class/hwmon/$HWMON/temp2_auto_point2_temp"
$SSH "cat /sys/class/hwmon/$HWMON/temp2_auto_point3_temp"
```

Warn: thresholds reset on reboot unless persisted.

---

#### action=all-runtime

Run in sequence: `status-runtime` → ask user which runtime config to apply.

---

### Step 4 — Persistence (optional)

To persist manual mode or custom thresholds across reboots, create a systemd one-shot service:

```bash
$SSH "cat > /etc/systemd/system/fan-controller.service << 'EOF'
[Unit]
Description=Fan Controller Initialization
After=multi-user.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/sh -c '\
  HWMON=\$(for d in /sys/class/hwmon/hwmon*; do \
    name=\$(cat \"\$d/name\" 2>/dev/null); \
    [ \"\$name\" = \"amc6821\" ] && echo \"\${d##*/}\" && break; \
  done); \
  HWMON=\${HWMON:-hwmon0}; \
  BASE=/sys/class/hwmon/\$HWMON; \
  echo 2 > \$BASE/pwm1_enable; \
  echo 0     > \$BASE/temp2_auto_point1_temp; \
  echo 48000 > \$BASE/temp2_auto_point2_temp; \
  echo 58000 > \$BASE/temp2_auto_point3_temp'

[Install]
WantedBy=multi-user.target
EOF"

$SSH "systemctl daemon-reload && systemctl enable fan-controller.service"
```

Ask the user: "Enable and start the persistence service now? (yes/no)"
If yes:
```bash
$SSH "systemctl start fan-controller.service"
$SSH "systemctl status fan-controller.service"
```

---

### Step 5 — Report results

After every action:
1. Show the exact commands run and their output.
2. For source changes: list every file modified and what was changed.
3. For runtime changes: display before/after sysfs values.
4. Warn about non-persistent settings.
5. Offer next steps:

| Action taken | Suggested next step |
|---|---|
| `enable-dts` | Run `kernel-config`, then rebuild with `kas-container build` |
| `kernel-config` | Run `enable-dts` if not done, then rebuild |
| `all-source` | Rebuild: `cd $source_dir && ./kas-container build meta-qcom/ci/iq-9075-evk.yml:meta-qcom/ci/qcom-distro.yml:meta-qcom/ci/linux-qcom-6.18.yml:meta-qcom/ci/lock.yml` |
| `auto-mode` | Run `status-runtime` to verify RPM |
| `manual-mode` | Run `status-runtime` to verify RPM |
| `set-thresholds` | Run `auto-mode` then `status-runtime` |
| Any runtime change | Offer to create persistence service (Step 4) |

---

## Reference

### AMC6821 sysfs node summary

| Node | Read/Write | Description |
|------|-----------|-------------|
| `name` | R | Should read `amc6821` |
| `pwm1_enable` | R/W | `1` = manual, `2` = automatic |
| `pwm1` | R/W | Fan power 0–255 |
| `fan1_input` | R | Fan speed in RPM |
| `temp2_input` | R | Remote temperature sensor in milli°C |
| `temp2_auto_point1_temp` | R/W | Fan-off threshold in milli°C |
| `temp2_auto_point2_temp` | R/W | Minimal-speed threshold in milli°C |
| `temp2_auto_point3_temp` | R/W | Full-speed threshold in milli°C |

### Default auto-mode thresholds (IQ-9075 EVK)

| Threshold | Default | Fan behavior |
|---|---|---|
| point1 | 0 °C (0 milli°C) | Fan off |
| point2 | 48 °C (48000 milli°C) | Minimum speed (~PWM 85/255) |
| point3 | 58 °C (58000 milli°C) | Full speed (PWM 255/255) |

### Source files summary (QLI 2.0 / meta-qcom)

| File | Purpose |
|------|---------|
| `meta-qcom/recipes-kernel/linux/linux-qcom_6.18.bb` | Kernel recipe — add patch to SRC_URI |
| `meta-qcom/recipes-kernel/linux/linux-qcom-6.18/configs/bsp-additions.cfg` | Kernel config fragment — add `CONFIG_SENSORS_AMC6821=y` |
| `meta-qcom/recipes-kernel/linux/linux-qcom-6.18/0002-...-Enable-AMC6821-fan.patch` | DTS patch — enable I2C19 + amc6821 node |
| `meta-qcom/conf/machine/iq-9075-evk.conf` | Machine conf — verify `lemans-evk-staging.dtbo` is listed |
| `arch/arm64/boot/dts/qcom/qcs9075-iq-9075-evk.dts` | Kernel DTS (modified via patch) |

### Build command (after source changes)

```bash
cd $source_dir
./kas-container build \
    meta-qcom/ci/iq-9075-evk.yml:meta-qcom/ci/qcom-distro.yml:meta-qcom/ci/linux-qcom-6.18.yml:meta-qcom/ci/lock.yml
```

Where `$source_dir` is the root of the user's QLI 2.0 checkout.

### Supported Platforms

| Platform | Chip | Fan Controller | Interface | GPIO |
|----------|------|----------------|-----------|------|
| IQ-9075 EVK | QCS9075 (Lemans) | TI AMC6821SQDBQRQ1 | I2C19 (QUP2 SE5) | GPIO99 (SDA), GPIO100 (SCL) |
