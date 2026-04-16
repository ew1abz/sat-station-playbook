# Test Specification — Satellite Station Ansible Playbook

## Scope

Integration tests for `site.yml` (install) and `uninstall.yml` (teardown) against a
real Windows 10/11 host. Tests run from the WSL2 control node.

efore
— the station hardware may not be present in the test environment.

---

## Framework

**[Molecule](https://molecule.readthedocs.io/) 24.x** with the `delegated` driver.

The delegated driver skips container lifecycle (create/destroy) and runs directly
against the configured Windows host via WinRM. SSH is available as an out-of-band
channel (e.g. to recover if a failed run leaves WinRM in a broken state, or to
inspect raw host state independent of Ansible).

### Dependencies (WSL2)

```bash
pip install molecule ansible-lint --break-system-packages
```

No extra Molecule driver package is needed for `delegated`.

---

## Inventory

Two inventory files live under `tests/inventory/`:

**`tests/inventory/hosts.ini`** — WinRM (used by Molecule converge/verify):

```ini
[sat_station]
testhost ansible_host=<WINDOWS_IP>

[sat_station:vars]
ansible_connection=winrm
ansible_winrm_transport=basic
ansible_user=<USERNAME>
ansible_password=<PASSWORD>
ansible_port=5985
ansible_winrm_scheme=http
```

**`tests/inventory/hosts_ssh.ini`** — SSH (out-of-band / manual use):

```ini
[sat_station]
testhost ansible_host=<WINDOWS_IP>

[sat_station:vars]
ansible_connection=ssh
ansible_user=<USERNAME>
ansible_password=<PASSWORD>
ansible_shell_type=powershell
```

Credentials are never committed. Supply them via environment variables or
Ansible Vault and reference as `ansible_user={{ lookup('env','TEST_USER') }}`.

---

## Scenarios

### 1. `default` — full install + verify

**Molecule lifecycle:**

| Phase | Action |
| --- | --- |
| `create` | No-op (delegated driver) |
| `converge` | `ansible-playbook site.yml` |
| `idempotency` | Re-run `site.yml`; assert **0 tasks changed** |
| `verify` | Run `tests/molecule/default/verify.yml` |
| `cleanup` | Run `ansible-playbook uninstall.yml` |
| `destroy` | No-op (delegated driver) |

### 2. `uninstall` — teardown + verify clean state

Run manually after `default` completes (or independently if the host is already
provisioned):

```bash
ansible-playbook uninstall.yml -i tests/inventory/hosts.ini
ansible-playbook tests/verify_uninstall.yml -i tests/inventory/hosts.ini
```

---

## Verify: post-install (`tests/molecule/default/verify.yml`)

### winrm_baseline

| Check | How |
| --- | --- |
| WinRM responds | `ansible.builtin.ping` (implicit — Molecule fails if unreachable) |
| Chocolatey is installed | `win_stat path: C:\ProgramData\chocolatey\bin\choco.exe` |

### hamlib

| Check | How |
| --- | --- |
| `C:\hamlib\bin\rigctld.exe` exists | `win_stat` |
| `C:\hamlib\bin\rigctl.exe` exists | `win_stat` |
| hamlib is in system PATH | `win_shell: where.exe rigctld.exe` succeeds |
| `rigctld-ft897` service exists and is running | `win_service_info` + assert `state == "running"` |
| `rigctld-ftx1` service exists and is running | `win_service_info` + assert `state == "running"` |
| Port 4532 is listening on 127.0.0.1 | `win_shell: netstat -ano \| findstr "127.0.0.1:4532.*LISTENING"` |
| Port 4535 is listening on 127.0.0.1 | `win_shell: netstat -ano \| findstr "127.0.0.1:4535.*LISTENING"` |
| `rigctld-ft897.bat` exists | `win_stat path: C:\hamlib\rigctld-ft897.bat` |
| `rigctld-ftx1.bat` exists | `win_stat path: C:\hamlib\rigctld-ftx1.bat` |
| Logs directory exists | `win_stat path: C:\hamlib\logs` |

### gpredict

| Check | How |
| --- | --- |
| `gpredict.exe` exists | `win_stat path: C:\Program Files (x86)\Gpredict\gpredict.exe` |
| `gpredict.cfg` exists | `win_stat` using `ansible_env.APPDATA` |
| `ft897.rig` exists in hwconf | `win_stat` |
| `ftx1.rig` exists in hwconf | `win_stat` |
| `rotator.rot` exists in hwconf | `win_stat` |
| `gpredict.cfg` contains operator callsign | `win_shell: Select-String` |
| `ft897.rig` contains port 4532 | `win_shell: Select-String` |
| `ftx1.rig` contains port 4535 | `win_shell: Select-String` |
| `rotator.rot` contains rotator IP | `win_shell: Select-String` |

### tle_update

| Check | How |
| --- | --- |
| `GpredictTLEUpdate` scheduled task exists and is enabled | `win_scheduled_task_info` + assert `state == "present"` and `enabled == true` |
| `Update-TLE.ps1` script exists | `win_stat path: C:\hamlib\scripts\Update-TLE.ps1` |
| Satellite data was seeded (at least one `.tle` file in satdata) | `win_find path: %APPDATA%\Gpredict\satdata pattern: "*.tle"` — assert `matched > 0` |

### net_adapter

| Check | How |
| --- | --- |
| USB Ethernet adapter has IP 192.168.1.1 | `win_shell: Get-NetIPAddress \| Where-Object IPAddress -eq '192.168.1.1'` — assert output non-empty |
| DHCP is disabled on that adapter | `win_shell: Get-NetIPInterface ... \| Select-Object Dhcp` — assert `Disabled` |

---

## Verify: post-uninstall (`tests/verify_uninstall.yml`)

### hamlib

| Check | How |
| --- | --- |
| `rigctld-ft897` service does not exist | `win_service_info` — assert `services \| length == 0` |
| `rigctld-ftx1` service does not exist | `win_service_info` — assert `services \| length == 0` |
| Port 4532 is NOT listening | `win_shell: netstat` — assert no match |
| Port 4535 is NOT listening | `win_shell: netstat` — assert no match |
| `C:\hamlib` directory does not exist | `win_stat` — assert `exists == false` |
| hamlib NOT in system PATH | `win_shell: where.exe rigctld.exe` — assert rc != 0 |

### gpredict

| Check | How |
| --- | --- |
| `gpredict.exe` does not exist | `win_stat` — assert `exists == false` |
| `gpredict.cfg` does not exist | `win_stat` — assert `exists == false` |
| `ft897.rig` does not exist | `win_stat` — assert `exists == false` |
| `ftx1.rig` does not exist | `win_stat` — assert `exists == false` |
| `rotator.rot` does not exist | `win_stat` — assert `exists == false` |

### tle_update

| Check | How |
| --- | --- |
| `GpredictTLEUpdate` task does not exist | `win_scheduled_task_info` — assert `task_exists == false` |

### net_adapter

| Check | How |
| --- | --- |
| DHCP re-enabled on USB Ethernet adapter | `win_shell: Get-NetIPInterface ... \| Select-Object Dhcp` — assert `Enabled` |
| Static IP 192.168.1.1 is gone | `win_shell: Get-NetIPAddress` — assert no match |

---

## Idempotency Criteria

Molecule's built-in idempotency check re-runs `site.yml` after a successful converge
and fails if any task reports `changed`. Expected result: **0 changed, 0 failed**.

Known non-idempotent tasks to watch:
- `Run TLE update immediately` uses `changed_when: false` — already suppressed.
- `net_adapter` static-IP task is gated by a pre-check — should be idempotent.

---

## Verify: hardware (`tests/verify_hardware.yml`)

Run manually after `site.yml` with all hardware connected and powered.
Requires: FT-897 on USB, FTX-1 on USB, polar-pilot reachable on 192.168.1.200.

```bash
ansible-playbook tests/verify_hardware.yml -i tests/inventory/hosts.ini
```

All checks use `rigctl.exe` / `rotctl.exe` from `C:\hamlib\bin\` on the Windows host.

### FT-897 uplink (port 4532)

| Check | Pass condition |
| --- | --- |
| `rigctl -m 2 -r 127.0.0.1:4532 f` exits 0 | rc == 0 |
| Returned frequency is a positive number | `float(stdout) > 0` |

### FTX-1 downlink (port 4535)

| Check | Pass condition |
| --- | --- |
| `rigctl -m 2 -r 127.0.0.1:4535 f` exits 0 | rc == 0 |
| Returned frequency is a positive number | `float(stdout) > 0` |

### polar-pilot rotator (192.168.1.200:4533)

| Check | Pass condition |
| --- | --- |
| `rotctl -m 2 -r 192.168.1.200:4533 p` exits 0 | rc == 0 |
| Azimuth is within 0–360° | `0 <= az <= 360` |
| Elevation is within 0–90° | `0 <= el <= 90` |
| Park command `P 0 0` is acknowledged | rc == 0 |
| Read-back after 3 s settle matches Az=0 El=0 | az == 0.0 and el == 0.0 |

The park round-trip is the key integration check — it exercises the full path from
Ansible (control node) → WinRM → Windows host → USB Ethernet → polar-pilot TCP → rotctld
protocol → in-memory state → response. It will fail if polar-pilot is running the stub
build (motor control not yet implemented) with correct state tracking, or if the
USB Ethernet link is not up.

---

## Running the Tests

```bash
# Full default scenario (install → idempotency → verify → cleanup)
cd /path/to/SatStationSpec
molecule test

# Install only (leave host provisioned for manual inspection)
molecule converge

# Verify only (against already-provisioned host)
molecule verify

# Uninstall + verify clean state
ansible-playbook uninstall.yml -i tests/inventory/hosts.ini
ansible-playbook tests/verify_uninstall.yml -i tests/inventory/hosts.ini

# Hardware verification (radios + polar-pilot must be connected and powered)
ansible-playbook tests/verify_hardware.yml -i tests/inventory/hosts.ini
```

---

## Out of Scope

- **TLE content validity** — only presence of seeded files is checked, not TLE format.
- **Gpredict UI** — no headless GUI testing; hwconf key-name correctness requires a
  live Gpredict manual check (see CLAUDE.md outstanding items).
