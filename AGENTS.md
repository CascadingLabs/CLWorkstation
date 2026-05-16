# CLWorkstation Agent Guide

## Project overview

CLWorkstation is an Ansible playbook that installs a consistent per-user SSH toolkit on Linux servers without changing other users' environments.

## Read first

- this file
- `README.md`
- relevant role files under `roles/`

## Working rules

- Preserve per-user installation semantics; avoid broad system changes unless the role explicitly requires them.
- Keep playbook behavior idempotent.
- Use variables and role structure instead of hard-coded one-off commands.
- Treat `inventory/hosts.yml` as local-only data; do not commit real host inventories.
- Keep distro-specific behavior isolated where practical.

## Key commands

- `ansible-playbook site.yml --limit <hostname> --check --diff --ask-become-pass`
- `ansible-playbook site.yml`
- `ansible-playbook site.yml --tags shell`
- `molecule test`

## Repository map

- `site.yml` — top-level play
- `roles/` — common, tools, shell, neovim behavior
- `vars/` — distro and tool catalogs
- `group_vars/` — workstation defaults
- `molecule/default/` — container test scenario
- `inventory/` — example inventory plus ignored local host file

## Architecture and constraints

Control-node Ansible configures remote Linux hosts over SSH. Target installs live under the remote user's home directory. Preserve idempotence, role boundaries, and the sentinel-file/versioning model described in `README.md`.

## Validation

Use `molecule test` for broad behavior changes. Prefer targeted `--check --diff` dry-runs when changing playbook logic or variables.
