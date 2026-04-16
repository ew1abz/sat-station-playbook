# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains the Ansible playbook for provisioning a **Windows-based amateur radio satellite ground station**, along with the original specification ([sat_station_ansible_spec.md](sat_station_ansible_spec.md)) and the polar-pilot-clone firmware project ([polar-pilot-clone/](polar-pilot-clone/)).

**Architecture:**

- **Control node:** WSL2 (Ubuntu 22.04+) running Ansible
- **Managed node:** Windows 10/11, controlled via WinRM (loopback, HTTP, basic auth)
- **Radio daemons:** Two independent `rigctld` instances via NSSM Windows services (FT-897 uplink on :4532, FTX-1 downlink on :4535)
- **Rotator:** polar-pilot firmware device speaks rotctld protocol natively on :4533 — no local rotctld daemon needed
- **Tracking software:** Gpredict connects to both rigctld daemons and directly to polar-pilot on the network

The two-rigctld design (rather than split-mode) is intentional — FTX-1's VFO split is incompatible with Gpredict (hamlib issue #1972).

## File Structure

```text
inventory/hosts.ini
group_vars/all.yml
site.yml
requirements.yml
roles/winrm_baseline/tasks/main.yml
roles/hamlib/tasks/main.yml
roles/hamlib/templates/rigctld-ft897.bat.j2
roles/hamlib/templates/rigctld-ftx1.bat.j2
roles/gpredict/tasks/main.yml
roles/gpredict/templates/gpredict.cfg.j2
roles/gpredict/templates/ft897.rig.j2
roles/gpredict/templates/ftx1.rig.j2
roles/gpredict/templates/rotator.rot.j2
roles/tle_update/tasks/main.yml
roles/tle_update/templates/Update-TLE.ps1.j2
roles/net_adapter/tasks/main.yml      # USB Ethernet adapter for direct rotator link
polar-pilot-clone/                    # Rust embedded firmware (Embassy, W5500, rotctld)
```

## Commands

### Prerequisites (WSL2, one-time setup)

```bash
sudo apt update && sudo apt install ansible python3-pip -y
pip install pywinrm --break-system-packages
ansible-galaxy collection install ansible.windows community.windows
# Or with pinned versions:
ansible-galaxy collection install -r requirements.yml
```

### Run the playbook

```bash
ansible-playbook -i inventory/hosts.ini site.yml
```

### Verify after playbook runs (PowerShell on Windows)

```powershell
# Test FT-897 uplink rigctld
"C:\hamlib\bin\rigctl.exe" -m 2 -r 127.0.0.1:4532 f

# Test FTX-1 downlink rigctld
"C:\hamlib\bin\rigctl.exe" -m 2 -r 127.0.0.1:4535 f

# Test polar-pilot rotctld (direct network)
"C:\hamlib\bin\rotctl.exe" -m 2 -r 192.168.1.200:4533 p
```

## Key Variables (group_vars/all.yml)

| Variable | Default | Notes |
| --- | --- | --- |
| `ft897_com_port` | (auto) | Auto-detected via USB VID_0403 (FTDI); set manually if needed |
| `ft897_baud` | 38400 | FT-897 Menu 19 = CAT RATE (Menu 14 = BEEP VOL, unrelated) |
| `ft897_hamlib_model` | 1023 | Yaesu FT-897 |
| `ft897_rigctld_port` | 4532 | |
| `ftx1_com_port` | (auto) | Auto-detected via USB VID_10C4 Enhanced port (CP2105); set manually if needed |
| `ftx1_baud` | 38400 | |
| `ftx1_hamlib_model` | 1051 | Yaesu FTX-1 |
| `ftx1_rigctld_port` | 4535 | 4533 reserved for polar-pilot |
| `rotator_host` | 192.168.1.200 | polar-pilot IP (static fallback after 120 s DHCP timeout) |
| `rotator_port` | 4533 | polar-pilot rotctld port |
| `usb_eth_ip` | 192.168.1.1 | Host IP on dedicated USB Ethernet link to rotator |
| `gpredict_zip_url` | (versioned SF URL) | Gpredict 2.3.37 ZIP from SourceForge |
| `gpredict_version` | 2.3.37 | |
| `gpredict_install_dir` | C:\Gpredict | Extracted from ZIP (not an NSIS installer) |
| `hamlib_version` | 4.7.1 | |
| `hamlib_install_dir` | C:\hamlib | |

## Outstanding Items

1. **Gpredict hwconf key names** — Verify `.rig`/`.rot` template keys match a live Gpredict install: install manually, create interfaces via GUI, inspect `%APPDATA%\Gpredict\hwconf\`.
2. **Windows Firewall** — Loopback traffic is unblocked by default; add explicit rules if connectivity issues arise.
3. **Celestrak rate-limiting** — TLE seed may be blocked if the host IP is rate-limited by Celestrak. Run `Update-TLE.ps1` manually once the block lifts.
