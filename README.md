# sat-station-playbook

Ansible playbook that provisions a Windows-based amateur radio satellite ground station.

## What it does

Automates setup of a two-radio + rotator tracking station on Windows 10/11, controlled from WSL2 via Ansible over WinRM:

- **Uplink radio:** Yaesu FT-897 → `rigctld` NSSM service on port 4532 (COM port auto-detected)
- **Downlink radio:** Yaesu FTX-1 → `rigctld` NSSM service on port 4535 (COM port auto-detected)
- **Rotator:** [polar-pilot](https://github.com/ew1abz/polar-pilot) speaks rotctld natively — Gpredict connects directly to `192.168.1.200:4533` over a dedicated USB-Ethernet link
- **Tracking software:** Gpredict

The playbook also configures the USB-Ethernet adapter's static IP and auto-detects radio COM ports via USB Vendor ID at run time.

## Usage

1. Complete the [manual prerequisites](sat_station_ansible_spec.md#2-prerequisites-manual-steps--not-automated) (WinRM, drivers, USB-Eth adapter connected)
2. Copy `inventory/hosts.ini` and fill in your Windows username and password
3. Edit `group_vars/all.yml` — at minimum set your callsign and location
4. Run:

   ```bash
   ansible-galaxy collection install -r requirements.yml
   ansible-playbook -i inventory/hosts.ini site.yml
   ```

## Contents

- [`sat_station_ansible_spec.md`](sat_station_ansible_spec.md) — full specification, variable reference, and known limitations
- [`site.yml`](site.yml) — top-level playbook
- [`group_vars/all.yml`](group_vars/all.yml) — all operator-configurable variables
- [`roles/`](roles/) — `winrm_baseline`, `net_adapter`, `hamlib`, `gpredict`, `tle_update`
