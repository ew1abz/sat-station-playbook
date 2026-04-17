# Ansible Playbook Spec — Windows Satellite Tracking Station

**Version:** 1.0  
**Target OS:** Windows 10/11 (managed node)  
**Control node:** WSL2 (Ubuntu 22.04+) on the same physical laptop  
**Purpose:** Reproducible provisioning of a two-radio + rotator satellite ground station  
**Radios:** Yaesu FT-897 (uplink) · Yaesu FTX-1 (downlink)  
**Rotator:** polar-pilot firmware (rotctld protocol, TCP)  
**Tracking software:** Gpredict (Windows native)

---

## 1. Architecture Overview

```text
WSL2 (Ubuntu)
└── ansible-playbook → WinRM → Windows host (localhost)
                                  ├── NSSM service: rigctld-ft897   :4532  ← FT-897 on auto-detected COM
                                  ├── NSSM service: rigctld-ftx1    :4535  ← FTX-1 on auto-detected COM
                                  ├── USB-Eth adapter               192.168.1.1/24
                                  └── Gpredict.exe
                                        ├── Radio 1 (uplink)   → 127.0.0.1:4532
                                        ├── Radio 2 (downlink) → 127.0.0.1:4535
                                        └── Rotator            → 192.168.1.200:4533

polar-pilot (STM32L432 + W5500 Ethernet)   192.168.1.200:4533
└── speaks rotctld protocol natively — no local rotctld daemon needed
    (falls back to 192.168.1.200 after 120 s if no DHCP server responds)
```

The control node and managed node are the same physical machine. Ansible runs inside
WSL2 and manages Windows via WinRM over the loopback address. This is a deliberate
educational pattern — it scales identically to a remote managed node.

**Why two separate rigctld instances instead of one radio in split mode:**  
The FTX-1's VFO split implementation is incompatible with Gpredict's split commands
(hamlib issue #1972, Dec 2025). Using two independent daemons — one per radio, one
per direction — sidesteps this completely and is the architecturally correct approach
for a two-radio satellite station.

---

## 2. Prerequisites (Manual Steps — Not Automated)

These steps must be completed before running the playbook. They cannot be reliably
automated because they require physical interaction or one-time GUI actions.

### 2.1 WinRM on Windows

**WinRM (Windows Remote Management)** is Microsoft's implementation of the WS-Management protocol — the Windows equivalent of SSH for remote command execution. Ansible uses it as the transport layer when managing Windows hosts. It exposes an HTTP/HTTPS listener (default port 5985/5986) that accepts SOAP-encoded commands, which Ansible's `pywinrm` library translates from standard Ansible module calls. In this setup, Ansible in WSL2 connects to WinRM on the same physical machine via the loopback address, keeping all traffic local and avoiding network exposure.

Run in an elevated PowerShell on the Windows host:

```powershell
winrm quickconfig -q
winrm set winrm/config/service/auth '@{Basic="true"}'
winrm set winrm/config/service '@{AllowUnencrypted="true"}'
# For local-only use (loopback), plain HTTP is acceptable.
# For remote targets, use HTTPS with a certificate instead.
```

### 2.2 Ansible on WSL2

```bash
sudo apt update
sudo apt install ansible python3-pip -y
pip install pywinrm --break-system-packages
ansible-galaxy collection install ansible.windows community.windows
```

### 2.3 COM Port Drivers

Both radios must be plugged in before running the playbook. COM ports are
**auto-detected at run time** via USB Vendor ID — no manual entry in `group_vars` needed.

| Radio   | USB chip                                  | Driver source           |
|---------|-------------------------------------------|-------------------------|
| FT-897  | FTDI (CT-62 cable or USB-serial adapter)  | FTDI CDM driver         |
| FTX-1   | Silicon Labs CP2105 (built-in USB-C)      | Silabs CP210x driver    |

The FTX-1 CP2105 presents two COM ports. The playbook selects the Enhanced (CAT-1) port
automatically by matching `Enhanced` in the Windows PnP friendly name.

### 2.4 Hamlib Version Requirement

FTX-1 support was added in **hamlib 4.7**. Confirm the version downloaded (see section
5.2) is ≥ 4.7 before deploying. If the bundled hamlib inside the Gpredict Windows
installer is older, the standalone hamlib binaries (rigctld.exe, rotctld.exe) must be
sourced from the hamlib 4.7 release separately.

### 2.5 USB Ethernet Adapter and Rotator

Connect the dedicated USB-Ethernet adapter before running the playbook. The playbook
detects it by USB bus type and assigns it `192.168.1.1/24`.

polar-pilot must be powered and connected to the same adapter. It attempts DHCP for
120 s then falls back to `192.168.1.200/24`. With no DHCP server on the dedicated
adapter, the fallback engages automatically — no DHCP server is needed on the PC.

---

## 3. Repository Layout

```text
sat-station/
├── inventory/
│   └── hosts.ini
├── group_vars/
│   └── all.yml
├── roles/
│   ├── winrm_baseline/        # WinRM connection test + Chocolatey bootstrap
│   ├── net_adapter/           # USB-Eth adapter static IP configuration
│   ├── hamlib/                # COM auto-detection, hamlib install, rigctld NSSM services
│   ├── gpredict/              # Install Gpredict, deploy config + hwconf files
│   └── tle_update/            # Scheduled Task for periodic TLE refresh
├── requirements.yml           # Ansible collection pins
└── site.yml                   # Top-level playbook
```

---

## 4. Inventory and Variables

### `inventory/hosts.ini`

```ini
[sat_station]
windows_host ansible_host=127.0.0.1

[sat_station:vars]
ansible_connection=winrm
ansible_winrm_transport=basic
ansible_user=YOUR_WINDOWS_USERNAME
ansible_password=YOUR_WINDOWS_PASSWORD
ansible_port=5985
ansible_winrm_scheme=http
```

> **Note:** For production or remote use, replace basic/HTTP with Kerberos or
> certificate-based auth over HTTPS (port 5986).

### `group_vars/all.yml`

```yaml
# --- Operator identity ---
operator_callsign: "YOURCALL"
operator_gridsquare: "AA00aa"
operator_lat: 0.0          # decimal degrees, north positive
operator_lon: 0.0          # decimal degrees, east positive
operator_alt_m: 0          # metres above sea level

# --- Radio: FT-897 (uplink) ---
# ft897_com_port is auto-detected at run time via USB VID_0403 (FTDI).
# Override here only if auto-detection fails.
ft897_com_port: ""
ft897_baud: 38400           # FT-897 Menu 19 = CAT RATE (Menu 14 = BEEP VOL, unrelated)
ft897_hamlib_model: 1023    # hamlib model for Yaesu FT-897
ft897_rigctld_port: 4532

# --- Radio: FTX-1 (downlink) ---
# ftx1_com_port is auto-detected at run time via USB VID_10C4 + "Enhanced" port (CP2105).
# Override here only if auto-detection fails.
ftx1_com_port: ""
ftx1_baud: 38400
ftx1_hamlib_model: 1051     # Yaesu FTX-1
ftx1_rigctld_port: 4535     # 4533 is reserved for polar-pilot

# --- Rotator (polar-pilot) ---
# polar-pilot speaks rotctld protocol directly — no local rotctld daemon needed.
# Falls back to 192.168.1.200 after 120 s with no DHCP response.
rotator_host: "192.168.1.200"
rotator_port: 4533

# --- USB Ethernet adapter (dedicated link to rotator) ---
# Detected automatically by USB bus type. Must share a /24 with rotator_host.
usb_eth_ip: "192.168.1.1"
usb_eth_prefix: 24

# --- Hamlib ---
hamlib_version: "4.7.1"
hamlib_install_dir: 'C:\hamlib'
hamlib_zip_url: "https://github.com/Hamlib/Hamlib/releases/download/4.7.1/hamlib-w64-4.7.1.zip"

# --- Gpredict ---
# Gpredict ships as a portable ZIP (no NSIS installer). Extracted to gpredict_install_dir.
gpredict_zip_url: "https://downloads.sourceforge.net/project/gpredict/Gpredict/2.3.37/gpredict-win32-2.3.37.zip"
gpredict_version: "2.3.37"
gpredict_install_dir: 'C:\Gpredict'

# --- NSSM ---
nssm_chocolatey_pkg: "nssm"

# --- TLE sources ---
tle_sources:
  - name: "amateur"
    url: "https://celestrak.org/pub/TLE/amateur.txt"
  - name: "cubesat"
    url: "https://celestrak.org/pub/TLE/cubesat.txt"

tle_update_schedule_hour: 6
```

---

## 5. Roles

### 5.1 `winrm_baseline`

**Purpose:** Verify connectivity, bootstrap Chocolatey.

**Tasks:**

1. `win_ping` — verify WinRM connectivity
2. `win_chocolatey` with `state: present` and `bootstrap: true` — install Chocolatey if absent
3. Assert Chocolatey version ≥ 2.0

**No variables beyond connection vars.**

---

### 5.2 `net_adapter`

**Purpose:** Configure the dedicated USB-Ethernet adapter with a static IP so that
polar-pilot's fallback address is reachable immediately after boot.

**Tasks:**

1. Find the USB-Ethernet adapter using `Get-NetAdapter -Physical` filtered by
   `InterfaceDescription -like '*USB*'`. Skip with a warning if none is found
   (adapter may not be plugged in; role is re-run safely once connected).

2. Check whether `{{ usb_eth_ip }}` is already assigned to the adapter.

3. If not: remove any existing IPv4 addresses, disable DHCP on the adapter, and
   assign `{{ usb_eth_ip }}/{{ usb_eth_prefix }}` via `New-NetIPAddress`.

**Key variables:** `usb_eth_ip` (default `192.168.1.1`), `usb_eth_prefix` (default `24`).

---

### 5.3 `hamlib`

**Purpose:** Auto-detect radio COM ports, deploy hamlib Windows binaries, and register
`rigctld` instances as persistent Windows services using NSSM.

**What is NSSM?** NSSM (Non-Sucking Service Manager) is a utility that wraps any
executable — including console programs like `rigctld.exe` — as a proper Windows
service. Windows has no built-in equivalent of `systemd` or `launchd` for arbitrary
executables: the native Service Control Manager requires services to implement a
specific Win32 API. NSSM bridges that gap by acting as the SCM-compliant wrapper,
forwarding start/stop/restart signals and capturing stdout/stderr to a log file.
This gives `rigctld` automatic startup at boot and automatic restart on crash without
any changes to the hamlib binaries themselves.

**Tasks:**

1. **Auto-detect COM ports** via `Get-PnpDevice -Class Ports`:
   - FT-897: filter `InstanceId -like '*VID_0403*'` (FTDI chip)
   - FTX-1: filter `InstanceId -like '*VID_10C4*'` and `FriendlyName -like '*Enhanced*'`
     (CP2105 dual-port; Enhanced = CAT-1 port)
   - Extract port name from `FriendlyName` with regex; set as Ansible facts.
   - Fail fast with a descriptive error if either radio is not detected.

2. Install NSSM via Chocolatey.

3. Create `{{ hamlib_install_dir }}` and `{{ hamlib_install_dir }}\logs`.

4. Download hamlib ZIP from `{{ hamlib_zip_url }}` via `win_get_url`.

5. Extract ZIP to a staging directory; robocopy the versioned subfolder to
   `{{ hamlib_install_dir }}` (the ZIP contains a top-level versioned folder,
   e.g. `hamlib-w64-4.7\`, that must be stripped).

6. Add `{{ hamlib_install_dir }}\bin` to the system `PATH`.

7. Template launch scripts using the detected COM ports:
   - `rigctld-ft897.bat` → `{{ hamlib_install_dir }}\rigctld-ft897.bat`
   - `rigctld-ftx1.bat` → `{{ hamlib_install_dir }}\rigctld-ftx1.bat`

8. Register each as an NSSM service via `community.windows.win_nssm` and start it.

**Templates:**

`rigctld-ft897.bat.j2`:

```bat
@echo off
"{{ hamlib_install_dir }}\bin\rigctld.exe" ^
  -m {{ ft897_hamlib_model }} ^
  -r {{ ft897_com_port }} ^
  -s {{ ft897_baud }} ^
  -t {{ ft897_rigctld_port }} ^
  -vvv >> "{{ hamlib_install_dir }}\logs\rigctld-ft897.log" 2>&1
```

`rigctld-ftx1.bat.j2`:

```bat
@echo off
"{{ hamlib_install_dir }}\bin\rigctld.exe" ^
  -m {{ ftx1_hamlib_model }} ^
  -r {{ ftx1_com_port }} ^
  -s {{ ftx1_baud }} ^
  -t {{ ftx1_rigctld_port }} ^
  -vvv >> "{{ hamlib_install_dir }}\logs\rigctld-ftx1.log" 2>&1
```

---

### 5.4 `gpredict`

**Purpose:** Install Gpredict and deploy operator-specific configuration.

**Tasks:**

1. Check if `{{ gpredict_install_dir }}\gpredict.exe` exists (register result as
   `gpredict_installed`)

2. If not installed:
   - Download ZIP from `{{ gpredict_zip_url }}` via `win_get_url`
   - Extract ZIP to staging via `community.windows.win_unzip`
   - Robocopy the versioned subfolder to `{{ gpredict_install_dir }}`

3. Ensure Gpredict config directory exists:  
   `%APPDATA%\Gpredict` and `%APPDATA%\Gpredict\satdata`

4. Template and deploy `gpredict.cfg` to `%APPDATA%\Gpredict\gpredict.cfg`

5. Deploy any pre-seeded TLE files to `%APPDATA%\Gpredict\satdata\`

Radio and rotator connections are stored in `%APPDATA%\Gpredict\hwconf\` as separate
`.rig` and `.rot` files, not in `gpredict.cfg`. The role templates all three:

- `ft897.rig` → uplink radio, connects to `127.0.0.1:{{ ft897_rigctld_port }}`
- `ftx1.rig` → downlink radio, connects to `127.0.0.1:{{ ftx1_rigctld_port }}`
- `rotator.rot` → connects **directly to polar-pilot** at `{{ rotator_host }}:{{ rotator_port }}`

> **Note:** The `.rig`/`.rot` key names should be verified against a live Gpredict
> installation — extract working examples before finalising the templates.

---

### 5.5 `tle_update`

**Purpose:** Register a Windows Scheduled Task that fetches fresh TLE data daily.

**Tasks:**

1. Template a PowerShell script `Update-TLE.ps1` that downloads each URL in
   `{{ tle_sources }}` to `%APPDATA%\Gpredict\satdata\`:

   ```powershell
   $dest = "$env:APPDATA\Gpredict\satdata"
   Invoke-WebRequest -Uri "{{ item.url }}" -OutFile "$dest\{{ item.name }}.tle"
   ```

2. Deploy `Update-TLE.ps1` to `{{ hamlib_install_dir }}\scripts\`

3. Register Scheduled Task via `community.windows.win_scheduled_task`:

   ```yaml
   community.windows.win_scheduled_task:
     name: "GpredictTLEUpdate"
     description: "Daily TLE refresh for Gpredict satellite tracking"
     actions:
       - path: powershell.exe
         arguments: "-ExecutionPolicy Bypass -File {{ hamlib_install_dir }}\\scripts\\Update-TLE.ps1"
     triggers:
       - type: daily
         start_boundary: "2000-01-01T{{ tle_update_schedule_hour }}:00:00"
     run_level: limited
     state: present
     enabled: true
   ```

4. Run the script directly (synchronously) after creation to seed TLE files immediately:

   ```yaml
   win_shell: powershell.exe -ExecutionPolicy Bypass -File "{{ hamlib_install_dir }}\scripts\Update-TLE.ps1"
   ```

---

## 6. Top-level Playbook (`site.yml`)

```yaml
---
- name: Provision satellite tracking station
  hosts: sat_station
  gather_facts: true

  pre_tasks:
    - name: Verify Windows target
      ansible.builtin.assert:
        that: ansible_os_family == "Windows"
        fail_msg: "This playbook targets Windows only."

  roles:
    - winrm_baseline
    - net_adapter
    - hamlib
    - gpredict
    - tle_update

  # No post_tasks — service running-state depends on hardware being connected.
  # Use tests/verify_install.yml to check deployment and
  # tests/verify_hardware.yml to verify radio/rotator connectivity.
```

---

## 7. Known Limitations and Open Items

| Item | Detail |
| ---- | ------ |
| **Gpredict hwconf key names** | The `.rig`/`.rot` key names in `hwconf\` templates should be verified against a live Gpredict installation before use. |
| **Hamlib zip layout** | The hamlib Windows zip contains a versioned top-level folder (e.g. `hamlib-w64-4.7.1\`). The playbook extracts to a staging directory and copies via robocopy. |
| **Firewall** | `rigctld` listens on loopback (127.0.0.1); polar-pilot is reached via the dedicated USB-Eth adapter. Windows Firewall should not interfere, but add `win_firewall_rule` tasks if connections fail. |
| **FT-897 baud rate** | Set via FT-897 **Menu 19** (`CAT RATE`). Default is 9600; playbook expects 38400 — change the menu setting before running. Menu 14 is BEEP VOL (unrelated). |
| **USB-Eth subnet conflict** | polar-pilot's fallback is `192.168.1.200/24`. If the main LAN is also `192.168.1.x`, Windows will have two adapters on the same subnet and routing may be unreliable. Change polar-pilot's fallback IP (in firmware) or use a different subnet on the dedicated adapter. |
| **polar-pilot DHCP timeout** | polar-pilot waits 120 s for DHCP before falling back to its static IP. With no DHCP server on the dedicated adapter, Gpredict will not be able to reach the rotator for the first ~2 minutes after power-on. |
| **Celestrak rate-limiting** | TLE seed (`Update-TLE.ps1`) may be blocked if the host IP is rate-limited by Celestrak. Run the script manually once the block lifts; scheduled daily runs will resume normally. |

---

## 8. Testing the Stack

### 8.1 Automated tests (from WSL2)

The test suite lives in `tests/` and is driven by Molecule. Credentials are
supplied via environment variables:

```bash
export ANSIBLE_TEST_USER=Temp
export ANSIBLE_TEST_PASS=<password>
```

**Important:** `molecule test` runs a full install → verify → uninstall cycle.
After the cycle completes, hamlib and Gpredict are removed from the Windows host.
Re-run `molecule converge` before running hardware tests.

```bash
# Full cycle (install → verify → uninstall)
~/.local/bin/molecule test

# Install only (leaves host provisioned)
~/.local/bin/molecule converge

# Software verification (no hardware required)
~/.local/bin/ansible-playbook tests/verify_install.yml \
    -i tests/inventory/hosts.ini -e @group_vars/all.yml

# Hardware verification (all hardware must be connected and powered)
# Run molecule converge first if the host is not already provisioned.
~/.local/bin/ansible-playbook tests/verify_hardware.yml \
    -i tests/inventory/hosts.ini -e @group_vars/all.yml

# Uninstall + verify clean state
~/.local/bin/molecule test -s uninstall
```

### 8.2 Manual verification (PowerShell on Windows)

```powershell
# Test rigctld for FT-897 (uplink)
"C:\hamlib\bin\rigctl.exe" -m 2 -r 127.0.0.1:4532 f
# Expected: current frequency in Hz

# Test rigctld for FTX-1 (downlink)
"C:\hamlib\bin\rigctl.exe" -m 2 -r 127.0.0.1:4535 f
# Expected: current frequency in Hz

# Test polar-pilot directly (wait 120 s after power-on for DHCP timeout)
"C:\hamlib\bin\rotctl.exe" -m 2 -r 192.168.1.200:4533 p
# Expected: current azimuth and elevation
```

In Gpredict: open **Edit → Preferences → Interfaces**, confirm each radio points to
the correct localhost port and the rotator points to `192.168.1.200:4533`, then open
a satellite module, right-click a satellite, and select **Track** to verify live
Doppler correction engages on both radios and the rotator begins moving.

---

## 9. Ansible Collections Required

```bash
ansible-galaxy collection install ansible.windows
ansible-galaxy collection install community.windows
```

Or pin versions in `requirements.yml`:

```yaml
collections:
  - name: ansible.windows
    version: ">=2.0.0"
  - name: community.windows
    version: ">=2.0.0"
```

Install with: `ansible-galaxy collection install -r requirements.yml`
