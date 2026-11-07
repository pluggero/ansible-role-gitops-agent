# ansible-role-gitops-agent

[![CI](https://github.com/pluggero/ansible-role-gitops-agent/actions/workflows/ci.yml/badge.svg)](https://github.com/pluggero/ansible-role-gitops-agent/actions/workflows/ci.yml) [![Ansible Galaxy downloads](https://img.shields.io/ansible/role/d/pluggero/gitops_agent?label=Galaxy%20downloads&logo=ansible&color=%23096598)](https://galaxy.ansible.com/ui/standalone/roles/pluggero/gitops_agent)

Install `sops`, `age`/`age-keygen`, and a pinned `ansible-core` into a
dedicated venv, then run a systemd-timer-driven `ansible-pull` against a
Git repository so the host keeps itself up to date.

## Requirements

**Collections:**

- `community.general` (provides `pacman`, used only on Arch Linux)

**On the target host:**

- `/etc/gitops/environment`, `/etc/gitops/deploy_key`, and
  `/etc/gitops/age.key` must be provisioned externally (e.g. by
  cloud-init) before the timer does anything. Until then this role's
  installed units are inert — see [Operational Guarantees](#operational-guarantees).

## Role Variables

Available variables are listed below, along with default values (see
`defaults/main.yml`) and full type/required constraints (see
`meta/argument_specs.yml`).

### Required

| Variable                | Type | Description                                                                                                                                                                    |
| ----------------------- | ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `gitops_agent_repo_url` | str  | Git URL that ansible-pull clones/pulls from.                                                                                                                                   |
| `gitops_agent_playbook` | str  | Path within the checkout to the entry playbook, e.g. `gitops/site.yml`. The only coupling point to the gitops repo's layout — `dirname()` of this value locates `ansible.cfg`. |

### Optional

| Variable                            | Default                          | Description                                                                                   |
| ----------------------------------- | -------------------------------- | --------------------------------------------------------------------------------------------- |
| `gitops_agent_branch`               | `main`                           | Git branch/ref ansible-pull checks out.                                                       |
| `gitops_agent_poll_interval`        | `5min`                           | systemd timer `OnUnitActiveSec=` value.                                                       |
| `gitops_agent_checkout_dir`         | `/opt/gitops-checkout`           | Directory ansible-pull clones the repo into.                                                  |
| `gitops_agent_install_dir`          | `/opt/gitops-agent`              | Directory the dedicated ansible-core venv is installed into.                                  |
| `gitops_agent_environment_file`     | `/etc/gitops/environment`        | Externally provisioned readiness marker; path only, never read.                               |
| `gitops_agent_deploy_key_path`      | `/etc/gitops/deploy_key`         | Externally provisioned SSH deploy key; path only, never read.                                 |
| `gitops_agent_age_key_path`         | `/etc/gitops/age.key`            | Externally provisioned age key; path only, never read.                                        |
| `gitops_agent_known_hosts`          | GitHub's published SSH host keys | See [Known Hosts](#known-hosts).                                                              |
| `gitops_agent_verify_commit`        | `false`                          | Adds `--verify-commit` to ansible-pull. Requires an operator-provisioned GPG keyring.         |
| `gitops_agent_sops_version`         | `3.13.3`                         | Pinned sops version.                                                                          |
| `gitops_agent_age_version`          | `1.3.2`                          | Pinned age/age-keygen version.                                                                |
| `gitops_agent_ansible_core_version` | `2.19.15`                        | Pinned ansible-core version, used as `ansible-core=={{ gitops_agent_ansible_core_version }}`. |

Both checksums are verified dynamically (sops: published checksums file;
age: GitHub release API digest) — nothing to update by hand.

### Known Hosts

`gitops_agent_known_hosts` is templated into a dedicated
`/etc/gitops/known_hosts` file — never the system/user default — and used
only by the gitops-pull deploy key via `GIT_SSH_COMMAND`. The shipped
default is GitHub's own published ed25519/ecdsa/rsa host keys. Override it
if the git host isn't GitHub; `StrictHostKeyChecking=yes` is always used,
so blind/TOFU host-key acceptance never happens.

### Vault Variables

N/A. This role stores no secret content — `gitops_agent_deploy_key_path`
and `gitops_agent_age_key_path` are paths to material provisioned by an
external process (e.g. cloud-init); this role never reads their contents.

## Operational Guarantees

- `gitops-pull.service` has no `Restart=`. On failure, the wrapper script
  exits non-zero and does nothing else — no rollback, no `docker compose
down`. Whatever containers the last successful pull left running keep
  running untouched. Failure is only visible via `systemctl status
gitops-pull.service` / `journalctl -u gitops-pull.service`; the next
  retry is the next timer tick.
- Until `/etc/gitops/{environment,deploy_key,age.key}` all exist, the
  wrapper logs `gitops not yet configured by cloud-init, skipping` and
  exits 0 — this is what makes the installed units inert until cloud-init
  provisions them.

## Dependencies

None.

## Example Playbook

```yaml
- hosts: all
  roles:
    - role: pluggero.gitops_agent
      vars:
        gitops_agent_repo_url: "git@github.com:myorg/my-gitops-repo.git"
        gitops_agent_playbook: "site.yml"
```

Wiring this role into a server's base image playbook (e.g. after a
`docker` role, so the stack is ready before polling starts) is left to
the consuming playbook/repo — out of scope here.

## License

MIT

## Author Information

This role was created in 2026 by pluggero.
