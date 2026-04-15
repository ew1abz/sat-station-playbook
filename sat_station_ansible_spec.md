# Ansible Playbook Spec — Windows Satellite Tracking Station

**Version:** 1.0  
**Target OS:** Windows 10/11 (managed node)  
**Control node:** WSL2 (Ubuntu 22.04+) on the same physical laptop  
**Purpose:** Reproducible provisioning of a two-radio + rotator satellite ground station  
**Radios:** Yaesu FT-897 (uplink) · Yaesu FTX-1 (downlink)  
**Rotator protocol:** EasyComm (TCP)  
**Tracking software:** Gpredict (Windows native)

---

## 1. Architecture Overview

```
WSL2 (Ubuntu)
└── ansible-playbook → WinRM → Windows host (localhost)
                                  ├── NSSM service: rigctld-ft897   :4532
                                  ├── NSSM service: rigctld-ftx1    :4533
                                  ├── NSSM service: rotctld         :4534
                                  └── Gpredict.exe
                                        ├── Radio 1 (uplink)   → 127.0.0.1:4532
                                        ├── Radio 2 (downlink) → 127.0.0.1:4533
                                        └── Rotator            → 127.0.0.1:4534
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

### 2.3 COM Port Identification

Connect the radios one at a time and note the assigned COM port in Windows Device Manager.
Record these values — they go into `group_vars/all.yml` (see section 4).

| Radio   | USB chip          | Driver source           |
|---------|-------------------|-------------------------|
| FT-897  | FTDI (CT-62 cable or USB-serial adapter) | FTDI CDM driver, or via Chocolatey `vcredist140` |
| FTX-1   | Silicon Labs CP210x (built-in USB-C)     | Silabs CP210x driver    |

The FTX-1 presents **two** virtual COM ports (Enhanced + Standard). Use the Enhanced
COM port (CAT-1) for Gpredict/rigctld.

### 2.4 Hamlib Version Requirement

FTX-1 support was added in **hamlib 4.7**. Confirm the version downloaded (see section
5.2) is ≥ 4.7 before deploying. If the bundled hamlib inside the Gpredict Windows
installer is older, the standalone hamlib binaries (rigctld.exe, rotctld.exe) must be
sourced from the hamlib 4.7 release separately.

### 2.5 Rotator TCP Endpoint

Confirm the rotator controller's IP address and TCP port before deployment. The rotator
must already be powered, network-reachable, and responding to EasyComm commands.

---

## 3. Repository Layout

```
sat-station/
├── inventory/
│   └── hosts.ini
├── group_vars/
│   └── all.yml
├── roles/
│   ├── winrm_baseline/        # WinRM connection test + Chocolatey bootstrap
│   ├── hamlib/                # Install hamlib binaries, configure services via NSSM
│   ├── gpredict/              # Install Gpredict, deploy config files
│   └── tle_update/            # Scheduled Task for periodic TLE refresh
├── files/
│   ├── gpredict/
│   │   ├── gpredict.cfg       # Gpredict main config (templated)
│   │   └── satellites/        # Optional: pre-seeded TLE files
│   └── nssm/
│       ├── rigctld-ft897.xml  # NSSM service definition (templated)
│       ├── rigctld-ftx1.xml   # NSSM service definition (templated)
│       └── rotctld.xml        # NSSM service definition (templated)
├── templates/
│   ├── gpredict.cfg.j2
│   ├── rigctld-ft897.bat.j2
│   ├── rigctld-ftx1.bat.j2
│   └── rotctld.bat.j2
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
operator_alt_m: 0          # altitude in metres above sea level

# --- Radio: FT-897 (uplink radio) ---
ft897_com_port: "COM3"
ft897_baud: 9600            # FT-897 default; change if set otherwise on the radio
ft897_hamlib_model: 1021    # hamlib model number for FT-897
ft897_rigctld_port: 4532    # TCP port Gpredict connects to

# --- Radio: FTX-1 (downlink radio) ---
ftx1_com_port: "COM5"       # Enhanced COM port (CAT-1); confirm in Device Manager
ftx1_baud: 38400            # FTX-1 default CAT-1 rate; must match radio menu setting
ftx1_hamlib_model: 1060     # PLACEHOLDER — verify actual model number in hamlib 4.7
                             # Check: rigctld.exe --list | findstr FTX
ftx1_rigctld_port: 4533

# --- Rotator ---
rotator_host: "192.168.1.100"   # IP of the rotator controller (TCP)
rotator_port: 4600              # TCP port on the rotator controller
rotator_easycomm_model: 202     # 202 = EasyComm II; use 204 for EasyComm III if supported
rotctld_port: 4534              # TCP port rotctld listens on (Gpredict connects here)

# --- Hamlib ---
hamlib_version: "4.7"
hamlib_install_dir: 'C:\hamlib'
hamlib_zip_url: "https://github.com/Hamlib/Hamlib/releases/download/4.7/hamlib-w64-4.7.zip"
# Verify URL against actual 4.7 release tag at https://github.com/Hamlib/Hamlib/releases

# --- Gpredict ---
gpredict_installer_url: "https://sourceforge.net/projects/gpredict/files/latest/download"
gpredict_install_dir: 'C:\Program Files\Gpredict'
gpredict_config_dir: '%APPDATA%\Gpredict'   # expands to C:\Users\<user>\AppData\Roaming\Gpredict

# --- NSSM (service manager) ---
nssm_chocolatey_pkg: "nssm"

# --- TLE sources ---
tle_sources:
  - name: "amateur"
    url: "https://celestrak.org/SOCRATES/query.php?catalog=amateur"
  - name: "cubesat"
    url: "https://celestrak.org/SOCRATES/query.php?catalog=cubesat"
# Simpler alternative (direct TLE files):
#   - name: "amateur"
#     url: "https://celestrak.org/pub/TLE/catalog.txt"

tle_update_schedule_hour: 6    # Run TLE update daily at this hour (local time)
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

### 5.2 `hamlib`

**Purpose:** Deploy hamlib Windows binaries and register rigctld + rotctld as persistent
Windows services using NSSM.

**What is NSSM?** NSSM (Non-Sucking Service Manager) is a utility that wraps any
executable — including console programs like `rigctld.exe` — as a proper Windows
service. Windows has no built-in equivalent of `systemd` or `launchd` for arbitrary
executables: the native Service Control Manager requires services to implement a
specific Win32 API. NSSM bridges that gap by acting as the SCM-compliant wrapper,
forwarding start/stop/restart signals and capturing stdout/stderr to a log file.
This gives `rigctld` and `rotctld` automatic startup at boot and automatic restart
on crash without any changes to the hamlib binaries themselves.

**Tasks:**

1. Install NSSM via Chocolatey:
   ```yaml
   win_chocolatey:
     name: "{{ nssm_chocolatey_pkg }}"
     state: present
   ```

2. Create hamlib install directory `{{ hamlib_install_dir }}`

3. Download hamlib ZIP from `{{ hamlib_zip_url }}` to a temp path using `win_get_url`

4. Unarchive hamlib ZIP into `{{ hamlib_install_dir }}` using `win_unzip`  
   (target is `hamlib_install_dir\bin\rigctld.exe` and `rotctld.exe`)

5. Add `{{ hamlib_install_dir }}\bin` to system `PATH` via `win_path`

6. Template launch scripts:
   - `rigctld-ft897.bat` → `{{ hamlib_install_dir }}\rigctld-ft897.bat`
   - `rigctld-ftx1.bat` → `{{ hamlib_install_dir }}\rigctld-ftx1.bat`
   - `rotctld.bat` → `{{ hamlib_install_dir }}\rotctld.bat`

7. Register each as an NSSM service (three iterations or three explicit tasks):
   ```
   nssm install rigctld-ft897 <path-to-bat>
   nssm set rigctld-ft897 Start SERVICE_AUTO_START
   nssm start rigctld-ft897
   ```
   Use `win_service` or `community.windows.win_nssm` module if available.

8. Verify each service is running with `win_service_info`

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

`rotctld.bat.j2`:
```bat
@echo off
"{{ hamlib_install_dir }}\bin\rotctld.exe" ^
  -m {{ rotator_easycomm_model }} ^
  -r {{ rotator_host }}:{{ rotator_port }} ^
  -t {{ rotctld_port }} ^
  -vvv >> "{{ hamlib_install_dir }}\logs\rotctld.log" 2>&1
```

**Idempotency note:** Check whether the NSSM service already exists before calling
`nssm install`. Use `win_service_info` as a guard and conditionally run install.

---

### 5.3 `gpredict`

**Purpose:** Install Gpredict and deploy operator-specific configuration.

**Tasks:**

1. Check if `{{ gpredict_install_dir }}\gpredict.exe` exists (register result as
   `gpredict_installed`)

2. If not installed:
   - Download installer EXE via `win_get_url`
   - Run silent installer via `win_package`:
     ```yaml
     win_package:
       path: "C:\\Temp\\gpredict-setup.exe"
       arguments: "/S"   # Gpredict uses NSIS, /S = silent
       state: present
     ```

3. Ensure Gpredict config directory exists:  
   `%APPDATA%\Gpredict` and `%APPDATA%\Gpredict\satdata`

4. Template and deploy `gpredict.cfg` to `%APPDATA%\Gpredict\gpredict.cfg`

5. Deploy any pre-seeded TLE files to `%APPDATA%\Gpredict\satdata\`

**`gpredict.cfg.j2` — key sections to template:**

```ini
[GLOBAL]
version=2
[GROUND_STATION]
name={{ operator_callsign }}
lat={{ operator_lat }}
lon={{ operator_lon }}
alt={{ operator_alt_m }}
qth={{ operator_gridsquare }}

# Radio 1 — uplink (FT-897)
[RADIO_1]
host=127.0.0.1
port={{ ft897_rigctld_port }}
type=1    ; 1 = RIG_TYPE_OTHER (duplex capable)

# Radio 2 — downlink (FTX-1)
[RADIO_2]
host=127.0.0.1
port={{ ftx1_rigctld_port }}
type=1

# Rotator
[ROTATOR_1]
host=127.0.0.1
port={{ rotctld_port }}
```

> **Note:** The actual Gpredict config file format uses INI-like keys but the exact
> section names and keys should be verified against the installed version's
> `gpredict.cfg` schema. Gpredict stores radio and rotator settings in separate
> `.rig` and `.rot` files under `%APPDATA%\Gpredict\hwconf\`. The playbook should
> template those files instead of (or in addition to) `gpredict.cfg` for full
> hardware configuration. Extract working examples from a configured installation
> before writing the templates.

---

### 5.4 `tle_update`

**Purpose:** Register a Windows Scheduled Task that fetches fresh TLE data daily.

**Tasks:**

1. Template a PowerShell script `Update-TLE.ps1` that downloads each URL in
   `{{ tle_sources }}` to `%APPDATA%\Gpredict\satdata\`:
   ```powershell
   $dest = "$env:APPDATA\Gpredict\satdata"
   Invoke-WebRequest -Uri "{{ item.url }}" -OutFile "$dest\{{ item.name }}.txt"
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

4. Optionally fire the task immediately after creation to seed the TLE files:
   ```yaml
   win_command: schtasks /Run /TN "GpredictTLEUpdate"
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
    - hamlib
    - gpredict
    - tle_update

  post_tasks:
    - name: Confirm rigctld-ft897 is running
      ansible.windows.win_service_info:
        name: rigctld-ft897
      register: svc_ft897
    - ansible.builtin.assert:
        that: svc_ft897.services[0].state == "running"

    - name: Confirm rigctld-ftx1 is running
      ansible.windows.win_service_info:
        name: rigctld-ftx1
      register: svc_ftx1
    - ansible.builtin.assert:
        that: svc_ftx1.services[0].state == "running"

    - name: Confirm rotctld is running
      ansible.windows.win_service_info:
        name: rotctld
      register: svc_rotctld
    - ansible.builtin.assert:
        that: svc_rotctld.services[0].state == "running"
```

---

## 7. Known Limitations and Open Items

| Item | Detail |
|------|--------|
| **FTX-1 hamlib model number** | Marked as placeholder `1060` in vars. Run `rigctld.exe --list \| findstr -i FTX` after hamlib install to confirm. Update `ftx1_hamlib_model` accordingly. |
| **COM port assignment** | Windows assigns COM ports dynamically. The playbook cannot discover or fix these; the operator must populate `ft897_com_port` and `ftx1_com_port` manually after first-time hardware connection. |
| **FTX-1 dual COM ports** | The FTX-1 registers two virtual COM ports (CAT-1 Enhanced, CAT-2 Standard). Ensure `ftx1_com_port` points to the Enhanced port. |
| **Gpredict hwconf files** | Hardware (radio/rotator) config in Gpredict is stored in `%APPDATA%\Gpredict\hwconf\*.rig` and `*.rot` files, not in `gpredict.cfg`. The templates in role `gpredict` must be verified against a live install and extended to cover these files for full idempotent hardware config. |
| **Gpredict installer URL** | The SourceForge "latest download" redirect may change. Pin to a specific version URL (e.g. `gpredict-2.3.37-win32.exe`) once confirmed working. |
| **Hamlib zip layout** | The hamlib Windows zip has a versioned top-level folder (e.g. `hamlib-4.7\`). The `win_unzip` task should strip or account for that prefix when placing binaries. |
| **NSSM + BAT vs direct** | NSSM can invoke `rigctld.exe` directly (without a `.bat` wrapper) using its `AppParameters` setting. Using `.bat` wrappers adds one process layer but makes the command visible and editable without touching NSSM config. Either approach is valid; document the choice. |
| **Firewall** | `rigctld` and `rotctld` listen on loopback only (127.0.0.1) by default. Windows Firewall should not block loopback traffic, but if Gpredict fails to connect, verify no third-party firewall is intercepting. Add an explicit `win_firewall_rule` task if needed. |
| **FT-897 baud rate** | The FT-897 ships with CAT baud set to 9600. Verify via Menu 14 (`CAT RATE`). Mismatched baud is the most common cause of `rigctld` connection failure. |
| **EasyComm variant** | Confirm whether the rotator speaks EasyComm II (model 202) or EasyComm III (model 204). Some controllers advertise EasyComm but implement a subset. Test with `rotctld -m 202 -r <host>:<port>` and `rotctl -m 2 -r 127.0.0.1:4534 p` (get position) before deploying. |

---

## 8. Testing the Stack (Manual Verification After Playbook Run)

```powershell
# From Windows PowerShell or cmd — test rigctld for FT-897
"C:\hamlib\bin\rigctl.exe" -m 2 -r 127.0.0.1:4532 f
# Expected: current frequency in Hz

# Test rigctld for FTX-1
"C:\hamlib\bin\rigctl.exe" -m 2 -r 127.0.0.1:4533 f
# Expected: current frequency in Hz

# Test rotctld
"C:\hamlib\bin\rotctl.exe" -m 2 -r 127.0.0.1:4534 p
# Expected: current azimuth and elevation
```

In Gpredict: open **Edit → Preferences → Interfaces**, confirm each radio and rotator
entry points to the correct localhost port, then open a satellite module, right-click a
satellite, and select **Track** to verify live Doppler correction engages on both radios
and the rotator begins moving.

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
