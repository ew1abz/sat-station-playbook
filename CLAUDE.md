# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains a single specification document ([sat_station_ansible_spec.md](sat_station_ansible_spec.md)) that defines an Ansible playbook for provisioning a **Windows-based amateur radio satellite ground station**. The Ansible code itself has not been implemented yet — the spec defines exactly what to build.

**Architecture:**
- **Control node:** WSL2 (Ubuntu 22.04+) running Ansible
- **Managed node:** Windows 10/11, controlled via WinRM (loopback, HTTP, basic auth)
- **Radio daemons:** Two independent `rigctld` instances via NSSM Windows services (uplink FT-897 on :4532, downlink FTX-1 on :4533)
- **Rotator:** `rotctld` via NSSM on :4534 (EasyComm II/III protocol)
- **Tracking software:** Gpredict connects to all three daemons on localhost

The two-rigctld design (rather than split-mode) is intentional — FTX-1's VFO split is incompatible with Gpredict (hamlib issue #1972).

## File Structure to Implement

The spec defines these files that need to be created:

```
inventory/hosts.ini
group_vars/all.yml
site.yml
requirements.yml
roles/winrm_baseline/tasks/main.yml
roles/hamlib/tasks/main.yml
roles/hamlib/templates/*.bat.j2          # rigctld/rotctld launch scripts
roles/gpredict/tasks/main.yml
roles/gpredict/templates/gpredict.cfg.j2
roles/tle_update/tasks/main.yml
roles/tle_update/templates/Update-TLE.ps1.j2
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
"C:\hamlib\bin\rigctl.exe" -m 2 -r 127.0.0.1:4533 f

# Test rotctld
"C:\hamlib\bin\rotctl.exe" -m 2 -r 127.0.0.1:4534 p
```

## Key Variables (group_vars/all.yml)

| Variable | Default | Notes |
|---|---|---|
| `ft897_com_port` | COM3 | Operator must discover manually |
| `ft897_baud` | 9600 | Must match radio Menu 14 setting |
| `ft897_hamlib_model` | 1021 | Yaesu FT-897D |
| `ft897_rigctld_port` | 4532 | |
| `ftx1_com_port` | COM5 | Must use Enhanced/CAT-1 port, not Standard/CAT-2 |
| `ftx1_baud` | 38400 | |
| `ftx1_hamlib_model` | 1060 | **Placeholder — verify with `rigctld.exe --list \| findstr FTX`** |
| `ftx1_rigctld_port` | 4533 | |
| `rotator_easycomm_model` | 202 | EasyComm II; confirm vs 204 before deploy |
| `rotctld_port` | 4534 | |
| `hamlib_version` | 4.7 | |
| `hamlib_install_dir` | C:\hamlib | |

## Outstanding Items Requiring Operator Input

These 10 items from the spec require manual resolution before the playbook can be fully automated:

1. **FTX-1 hamlib model** — Model 1060 is a placeholder; verify with `rigctld.exe --list | findstr FTX`
2. **COM port assignment** — Cannot be reliably automated; operator must identify COM3/COM5 manually
3. **FTX-1 dual COM ports** — Use Enhanced (CAT-1), not Standard (CAT-2) port
4. **Gpredict hwconf templates** — `.rig`/`.rot` files in `%APPDATA%\Gpredict\hwconf\` need separate Jinja2 templates
5. **Gpredict installer URL** — SourceForge redirect may change; pin to a specific version
6. **Hamlib zip layout** — Contains versioned top-level folder; `win_unzip` needs path-strip handling
7. **NSSM + BAT vs direct invocation** — Either approach valid; choose and document
8. **Windows Firewall** — Loopback unblocked by default; add explicit rule if connectivity issues arise
9. **FT-897 baud rate** — Verify radio Menu 14 matches `ft897_baud` variable
10. **EasyComm variant** — Confirm model 202 vs 204 with actual rotator hardware
