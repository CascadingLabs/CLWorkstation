<p align="center">
  <a href="https://github.com/CascadingLabs/CLWorkstation">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="media/logo-dark.svg">
      <source media="(prefers-color-scheme: light)" srcset="media/logo-light.svg">
      <img src="media/logo-dark.svg" alt="CLWorkstation" width="200">
    </picture>
  </a>
</p>

<p align="center">
  <a href="https://discord.gg/UnqRNzFYjM"><img src="https://img.shields.io/badge/Discord-Join-c4a882?labelColor=2e2319&logo=discord&logoColor=white" alt="Discord"></a>
  <a href="https://opensource.org/licenses/Apache-2.0"><img src="https://img.shields.io/badge/License-Apache_2.0-c4a882?labelColor=2e2319" alt="License"></a>
</p>

# CLWorkstation

Ansible playbook that installs a consistent SSH toolkit on any Linux server (Arch or Ubuntu) **per-user** — no root required for the tools themselves. One command and a fresh server has `lazydocker`, `btop`, `nvim` (+ LazyVim), `lazygit`, `ripgrep`, `fd`, `fzf`, `bat`, `eza`, `zoxide`, `starship`, the **JetBrainsMono Nerd Font** (Omarchy's default, so prompts/icons render correctly), plus a tested bash alias set.

Nothing lands outside `$HOME`, so coworkers sharing the box are untouched.

## How it runs

CLWorkstation runs in Ansible's **remote mode**: Ansible lives on your laptop
(the *control node*) and configures servers over SSH. Targets need only SSH
access and Python — nothing is installed on them beyond the tools in the
playbook itself. There's no agent, no daemon, and no open port besides SSH.

## One-time control-node setup

```bash
uv tool install ansible-core
ansible-galaxy collection install -r requirements.yml
cp inventory/hosts.yml.example inventory/hosts.yml
$EDITOR inventory/hosts.yml    # add your servers
```

### Connection identity (`ansible.cfg`)

`ansible.cfg` ships with placeholder connection vars under `[all:vars]` —
replace them with the SSH user and private key Ansible should connect with:

```ini
[all:vars]
ansible_user=<whoami_user>                       # e.g. the output of `whoami` on the target
ansible_ssh_private_key_file=~/.ssh/<your_ssh_key>   # e.g. ~/.ssh/id_ed25519
```

These are control-node-local defaults; per-host overrides still belong in
`inventory/hosts.yml`.

### First time load, no ssh key

Ansible has two password auth methods: SSH key and password.
```bash
ansible-playbook site.yml --limit <host> --check --diff --ask-pass --ask-become-pass
# just make sure that that the host is in host.yml already
# I have my public key in roles/common/files/id_ed25519.pub, so make sure you don't want to use a different key
```

### Loading your key into ssh-agent

So Ansible (and plain `ssh`) can authenticate without retyping your passphrase
on every connection, start an agent for the session and add your key:

```

### Loading your key into ssh-agent

So Ansible (and plain `ssh`) can authenticate without retyping your passphrase
on every connection, start an agent for the session and add your key:

```bash
eval "$(ssh-agent -s)"          # start the agent, export its env vars
ssh-add -l                      # confirm it's loaded
```

`eval` is required because `ssh-agent -s` prints shell commands (the socket
path and PID) that must be evaluated in your current shell — running it without
`eval` just dumps the variables instead of setting them. The agent lives for
the shell session; add the two lines to your shell rc to make it automatic.

## Everyday use

```bash
# Dry run against one host:
ansible-playbook site.yml --limit <hostname> --check --diff --ask-become-pass

# Real run, all hosts:
ansible-playbook site.yml

# Just refresh the shell config:
ansible-playbook site.yml --tags shell
```

## What goes where on the target

| Path                                    | Contents                             |
|-----------------------------------------|--------------------------------------|
| `~/.local/bin/`                         | Static binaries (rg, fd, btop, …)    |
| `~/.local/share/nvim-linux-x86_64/`     | Full neovim install                  |
| `~/.local/share/fonts/`                 | Nerd Fonts (JetBrainsMono)           |
| `~/.config/nvim/`                       | LazyVim starter (only if absent)     |
| `~/.bashrc`                             | Thin rc that sources `~/.bashrc.d/*` |
| `~/.bashrc.d/{aliases,fzf,starship}.sh` | Templated shell fragments            |
| `~/.cache/clworkstation/`               | Sentinel files for idempotence       |

Your existing `~/.bashrc` is preserved once at `~/.bashrc.pre-clworkstation`.

## Repo structure

```
CLWorkstation/
├── ansible.cfg                 ← inventory, host-key checks, [all:vars] conn identity
├── site.yml                    ← top-level play: common → tools → fonts → shell → neovim
├── requirements.yml            ← ansible-galaxy collections
├── inventory/
│   ├── hosts.yml.example       ← copy to hosts.yml (gitignored)
│   └── hosts.yml               ← your servers, not committed
├── group_vars/workstations.yml ← feature toggles + path vars
├── vars/
│   ├── tools.yml               ← tool catalog: pinned versions + release URLs
│   ├── fonts.yml               ← Nerd Font catalog: pinned versions + release URLs
│   ├── Archlinux.yml           ← distro base packages
│   └── Debian.yml
├── roles/
│   ├── common/                 ← creates ~/.local/bin, ~/.config, …
│   ├── tools/                  ← downloads + installs binaries from tools.yml
│   ├── fonts/                  ← installs Nerd Fonts from fonts.yml into ~/.local/share/fonts
│   ├── shell/                  ← templates bashrc + .bashrc.d/*.sh
│   └── neovim/                 ← clones LazyVim starter if absent
├── molecule/default/           ← Arch + Ubuntu container tests (prepare → converge → verify)
└── media/                      ← logo assets (see Assets repo conventions)
```

## Pinning a new tool version

Edit `vars/tools.yml`, bump `version` and `url`, delete the matching sentinel
file under `~/.cache/clworkstation/` on the target, then rerun the playbook.

## Fonts

The `fonts` role installs Nerd Fonts per-user into `~/.local/share/fonts` and
refreshes the user font cache with `fc-cache` (when `fontconfig` is present).
The catalog lives in `vars/fonts.yml` and tracks the same Nerd Font Omarchy
uses by default — **JetBrainsMono Nerd Font**.

```bash
# Just (re)install fonts:
ansible-playbook site.yml --tags fonts
```

Pin a new release the same way as tools: bump `version`/`url` in
`vars/fonts.yml`, delete the matching `font-<name>-<version>.installed` sentinel
on the target, and rerun. Set `enable_nerd_fonts: false` in
`group_vars/workstations.yml` to skip the role entirely.

## Local testing

Molecule and the Docker driver no longer ship as a single package, and
`uv tool` puts each tool in its own venv. Install both into one env so
`molecule` can find `ansible-config` and the docker plugin at runtime:

```bash
uv tool install --force molecule \
  --with 'molecule-plugins[docker]' \
  --with docker \
  --with ansible
```

Then run from the **repo root** (not from `molecule/default/`), with the
molecule tool's bin on `PATH` so `ansible-config` resolves:

```bash
export PATH="$HOME/.local/share/uv/tools/molecule/bin:$PATH"
molecule test   # Arch + Ubuntu containers → converge → idempotence → verify
```

Useful sub-commands during development:

```bash
molecule converge          # apply once, leave containers running
molecule login -h cl-arch  # shell into a converged container
molecule verify            # re-run verify.yml only
molecule destroy           # tear down
```

A `prepare.yml` runs before `converge.yml` and bootstraps python on the bare
Arch image and refreshes apt + installs `git` on the Ubuntu image — both are
prerequisites that the playbook itself assumes a real workstation already has.

## Adding a new tool

1. Add an entry to `vars/tools.yml` with the release URL and the archive-relative path to the binary.
2. Add the binary name to `molecule/default/verify.yml`.
3. Run `molecule test` — if both distros come up green, you're done.

## Related projects

| Project        | Repo                                                                     |
|----------------|--------------------------------------------------------------------------|
| Cascading Labs | [github.com/CascadingLabs](https://github.com/CascadingLabs)             |
| Assets         | [github.com/CascadingLabs/Assets](https://github.com/CascadingLabs/Assets) |
| QScrape        | [github.com/CascadingLabs/QScrape](https://github.com/CascadingLabs/QScrape) |
| Yosoi          | [github.com/CascadingLabs/Yosoi](https://github.com/CascadingLabs/Yosoi) |
| VoidCrawl      | [github.com/CascadingLabs/VoidCrawl](https://github.com/CascadingLabs/VoidCrawl) |

## Community

- **Discord:** [discord.gg/UnqRNzFYjM](https://discord.gg/UnqRNzFYjM)
- **Support:** see [SUPPORT.md](SUPPORT.md)
- **Security:** see [SECURITY.md](SECURITY.md)
- **Code of Conduct:** see [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md)

## Contact

[contact@cascadinglabs.com](mailto:contact@cascadinglabs.com)

## License

Apache 2.0 — see [LICENSE](LICENSE).
