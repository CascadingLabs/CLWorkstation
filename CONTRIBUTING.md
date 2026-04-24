# Contributing to CLWorkstation

Thanks for your interest! CLWorkstation is an Ansible playbook for bootstrapping a consistent SSH toolkit on Linux servers. Contributions that add support for more distros, pin safer tool versions, or improve idempotence are all welcome.

## Clone & Setup

```bash
git clone https://github.com/CascadingLabs/CLWorkstation.git
cd CLWorkstation
uv tool install ansible-core
ansible-galaxy collection install -r requirements.yml
cp inventory/hosts.yml.example inventory/hosts.yml
```

**Prerequisites:**

| Tool          | Version      | Install                                           |
|---------------|--------------|---------------------------------------------------|
| uv            | >= 0.5       | [docs.astral.sh/uv](https://docs.astral.sh/uv/)   |
| ansible-core  | >= 2.17      | `uv tool install ansible-core`                    |
| Docker        | (for tests)  | distro package manager                            |

## Install pre-commit hooks

```bash
uvx prek install
```

[Prek](https://github.com/thesuperzapper/prek) is a Rust-based pre-commit runner. Hooks run `yamllint`, `ansible-lint`, `gitleaks`, and enforce conventional commit messages via `commitizen`. To run all hooks manually:

```bash
uvx prek run --all-files
```

## Testing

### Local container tests (Molecule)

```bash
uv tool install 'molecule[docker]' molecule-plugins
cd molecule/default
molecule test
```

`molecule test` spins up Arch and Ubuntu containers, runs `site.yml`, asserts idempotence (second run must have zero changed tasks), then runs `verify.yml` to check each binary exists and the shell aliases resolve.

### Smoke test on a real server

```bash
ansible-playbook site.yml --limit <host> --check --diff    # dry run
ansible-playbook site.yml --limit <host>                   # real
ssh <host> -- 'which lazydocker btop nvim && bash -ic "alias gs"'
```

## Adding a tool

1. Add an entry to `vars/tools.yml` with the pinned version and GitHub release URL.
2. Add the binary name to the `loop` in `molecule/default/verify.yml`.
3. If the tool needs shell integration (completions, init eval), add a fragment under `roles/shell/templates/` and source it from `bashrc.j2` or a new `.bashrc.d/*.sh.j2`.
4. Run `molecule test` — both distros must stay green, including the idempotence check.

## Adding a distro

1. Add `vars/<OSFamily>.yml` (e.g. `vars/RedHat.yml`) with any distro-specific base packages.
2. Add a platform entry to `molecule/default/molecule.yml`.
3. `molecule test` — fix any role tasks that assumed a particular package manager.

## Issues

Pick the [template](https://github.com/CascadingLabs/CLWorkstation/issues/new/choose) that fits: Bug Report, Feature Request, Question, or Ticket. Blank issues are disabled so we always have the context we need.

## Pull Request Rules

1. **Branch from `main`** — use `feat/...`, `fix/...`, `docs/...`.
2. **Keep PRs focused** — one logical change per PR.
3. **Pass CI** — `yamllint`, `ansible-lint`, and `molecule test` must succeed.
4. **Fill in the PR template** — Intent, Changes, GenAI Usage, Risks.
5. **Link an issue** — reference the issue with `Closes #<number>`.

### Commit Conventions

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
feat: add role for installing docker engine
fix: correct neovim symlink target on Arch
docs: document --tags usage in README
```

## License

Contributions are licensed under Apache-2.0, matching the project.
