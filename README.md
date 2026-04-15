# SatStationSpec

Specification for an Ansible playbook that provisions a Windows-based amateur radio satellite ground station.

## What it does

Automates setup of a two-radio + rotator tracking station on Windows 10/11, controlled from WSL2 via Ansible over WinRM:

- **Uplink radio:** Yaesu FT-897 → `rigctld` service on port 4532
- **Downlink radio:** Yaesu FTX-1 → `rigctld` service on port 4533
- **Rotator:** EasyComm (TCP) → `rotctld` service on port 4534
- **Tracking software:** Gpredict connects to all three on localhost

Services are managed by NSSM so they start at boot and restart on failure.

## Contents

- [`sat_station_ansible_spec.md`](sat_station_ansible_spec.md) — full playbook specification including roles, variables, templates, and known limitations

The Ansible playbook itself is not yet implemented. The spec defines everything needed to build it.

## Status

Design phase. See the [open items](sat_station_ansible_spec.md#7-known-limitations-and-open-items) in the spec for what needs to be resolved before implementation.
