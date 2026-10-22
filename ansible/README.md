# Ansible playbooks

## Run locally on a workstation

The `workstations` inventory group contains `neptune` and `liquid-echo`. Run
workstation playbooks directly on each machine, using a local connection rather
than SSH.

### Prerequisites

- Check out this repository on the workstation.
- Have Ansible and Python available locally (`mise.toml` declares the Ansible tool).
- Run as your normal user with permission to use `sudo`; the playbooks use
  `become: true` for privileged tasks.
- Have the services and packages being configured already installed. These
  playbooks configure them rather than install them.

### Target the current workstation

Run commands from this repository's `ansible/` directory so Ansible loads
`ansible.cfg` and the `hosts` inventory:

```bash
cd /path/to/docker-compose/ansible
```

Use `--limit "$(hostname)"` to select the current workstation; `hostname`
returns its short inventory name (`neptune` or `liquid-echo`). Always combine
this limit with `--connection local`. A local connection runs tasks on the
machine executing Ansible, but does not change which inventory hosts are
selected. Without the limit, the same tasks could run locally multiple times
under different inventory host names.

### Preview and apply

Preview changes with check mode (a dry run, subject to module support):

```bash
ansible-playbook nfs-mount.yml --connection local --limit "$(hostname)" --ask-become-pass --check --diff
ansible-playbook docker-daemon.yml --connection local --limit "$(hostname)" --ask-become-pass --check --diff
```

Apply the configuration by omitting `--check --diff`:

```bash
ansible-playbook nfs-mount.yml --connection local --limit "$(hostname)" --ask-become-pass
ansible-playbook docker-daemon.yml --connection local --limit "$(hostname)" --ask-become-pass
```

`--ask-become-pass` prompts for your local sudo password; omit it if your account
has passwordless sudo.

### Troubleshooting sudo-rs on liquid-echo

On `liquid-echo`, sudo-rs can produce a password prompt that Ansible does not
recognize, causing this error during fact gathering:

```text
Timed out waiting for become success or become password prompt.
```

Use the traditional sudo implementation, `sudo.ws`, for the Ansible invocation:

```bash
ansible-playbook nfs-mount.yml --connection local --limit "$(hostname)" --ask-become-pass --extra-vars 'ansible_become_exe=sudo.ws'
```

This workaround has been verified on `liquid-echo`. Add the same
`--extra-vars 'ansible_become_exe=sudo.ws'` option to other playbook commands,
including check-mode runs, when needed. It requires `sudo.ws` to be installed
(check with `command -v sudo.ws`) and does not change the system's default sudo
implementation or require passwordless sudo.

The Python interpreter discovery warning is separate from this failure.
See the [upstream Ansible issue](https://github.com/ansible/ansible/issues/85837)
for details.

## Playbook scope

| Playbook | Inventory target | Effect |
| --- | --- | --- |
| `nfs-mount.yml` | `workstations` | Adds an NFS entry for `/evil_ripple` to `/etc/fstab`; does not mount it automatically (`noauto`). |
| `docker-daemon.yml` | `all` | Writes `/etc/docker/daemon.json`; does not restart Docker. |
| `cron_zpool_status.yml` | `node1` | Configures the ZFS status cron job. |
| `samba.yml` | `node1` | Configures Samba shares. |
| `sshd-docker.yml` | `node1` | Configures SSH connection limits and restarts SSH when changed. |
| `timesyncd.yml` | `node1` | Stops and disables systemd-timesyncd. |

Only `nfs-mount.yml` and `docker-daemon.yml` currently apply to workstations.
Limiting a `node1`-only playbook to a workstation will select no hosts; a local
connection does not override a playbook's inventory target.
