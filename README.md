# sat-station-playbook

Ansible playbook that provisions a Windows-based amateur radio satellite ground station.

## What it does

Automates setup of a two-radio + rotator tracking station on Windows 10/11, controlled from WSL2 via Ansible over WinRM:

- **Uplink radio:** Yaesu FT-897 → `rigctld` service on port 4532
- **Downlink radio:** Yaesu FTX-1 → `rigctld` service on port 4533
- **Rotator:** EasyComm (TCP) → `rotctld` service on port 4534
- **Tracking software:** Gpredict connects to all three on localhost

Services are managed by NSSM so they start at boot and restart on failure.

## Usage

1. Complete the [manual prerequisites](sat_station_ansible_spec.md#2-prerequisites-manual-steps--not-automated) (WinRM, COM ports, hamlib version)
2. Copy `inventory/hosts.ini` and fill in your Windows username and password
3. Edit `group_vars/all.yml` — at minimum set your callsign, location, and COM ports
4. Run:

   ```bash
   ansible-galaxy collection install -r requirements.yml
   ansible-playbook -i inventory/hosts.ini site.yml
   ```

## Contents

- [`sat_station_ansible_spec.md`](sat_station_ansible_spec.md) — full specification, variable reference, and known limitations
- [`site.yml`](site.yml) — top-level playbook
- [`group_vars/all.yml`](group_vars/all.yml) — all operator-configurable variables
- [`roles/`](roles/) — `winrm_baseline`, `hamlib`, `gpredict`, `tle_update`
