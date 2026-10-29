# Rootless Docker Ansible Role

Installs Docker CE from the upstream apt repository and runs it **rootless**: the
system-wide daemon is stopped and disabled, and an unprivileged user runs its own
daemon as a systemd user unit.

## Requirements

- Ubuntu >= 24.04  
  Debian >= 12
- `x86_64`, `aarch64`, `armv7l`, `ppc64le` or `s390x` (Docker publishes an apt repository for these)
- Ansible >= 2.15 (`deb822_repository`)
- `ansible.posix` collection (sysctl)
- Root privileges (`become: true`)

## Role Variables

Defined in `defaults/main.yml`:

| Variable | Default | |
|---|---|---|
| `docker_service_user` | `service` | user the rootless daemon runs as |
| `docker_service_group` | `service` | |
| `docker_service_home` | `/home/{{ docker_service_user }}` | |
| `docker_service_shell` | `/bin/bash` | |
| `docker_unprivileged_port_start` | `0` | lowest port the daemon may publish; `0` allows 80/443 |
| `docker_max_user_namespaces` | `15000` | `user.max_user_namespaces` where AppArmor does not scope user namespaces |
| `docker_apt_repo_url` | `https://download.docker.com/linux/<distro>` | `ubuntu` or `debian` |
| `docker_apt_key_url` | `https://download.docker.com/linux/<distro>/gpg` | |
| `docker_packages_state` | `latest` | `latest` or `present`, see below |
| `docker_packages` | `docker-ce`, `-cli`, `-rootless-extras`, `containerd.io`, `-buildx-plugin`, `-compose-plugin` | |
| `docker_daemon_config` | `{}` | contents of the service user's `~/.config/docker/daemon.json` |
| `docker_setuptool_args` | `[]` | extra arguments for `dockerd-rootless-setuptool.sh install` |

The role keeps the `docker_*` prefix rather than `docker_rootless_*`: the vocabulary
is shared with a runner role that mounts this daemon's socket, and prefixing it per
role would break the one thing they have in common.

The service user gets whatever uid, gid and subordinate id range the system
allocates, so a host where a fixed id is already taken does not fail.

## Example Playbook

```yaml
- hosts: docker
  become: true
  roles:
    - role: docker_rootless
      docker_daemon_config:
        default-address-pools:
          - base: 172.26.128.0/17
            size: 24
```

A consumer that adds a runtime (GPU hosts, for example) passes it through
`docker_daemon_config`; this role is the only writer of `daemon.json`.

## Docker package upgrades

`docker_packages_state` defaults to `latest`, so **a run upgrades Docker CE**. Set it
to `present` to freeze a host where it is.

An upgrade replaces the `dockerd` binary on disk but does not restart the daemon —
that would stop every container on the host, including whatever is published on it.
The role therefore reports the mismatch and leaves the restart to a maintenance
window:

```
Docker CE is now 29.x, but the rootless daemon is still running 27.y ...
```

Run as the service user:

```sh
sudo -iu service
systemctl --user restart docker
```

A frozen host (`present`) still reports this drift, because the check compares the
running daemon to the installed binary rather than watching for a change. A warning
on a frozen host is not a broken run.

## Usage on the host

The daemon listens only on the service user's socket:

```sh
sudo -iu service
docker ps
```

No `DOCKER_HOST` is needed: `dockerd-rootless-setuptool.sh` creates a `rootless`
docker context pointing at `unix:///run/user/<uid>/docker.sock`.

The role exports `XDG_RUNTIME_DIR` from the service user's `.bashrc` because
`sudo -iu` gives a login shell but not a login session, and without it
`systemctl --user` fails with `Failed to connect to bus: No medium found`.

## Unprivileged user namespaces

Where AppArmor restricts unprivileged user namespaces (Ubuntu 24.04+), the role installs
`/etc/apparmor.d/usr.bin.rootlesskit`, and fails if the host lacks apparmor >= 4.0.

Elsewhere it sets `user.max_user_namespaces={{ docker_max_user_namespaces }}` and, where
present, `kernel.unprivileged_userns_clone=1`.

## Tests

`molecule test` runs on Ubuntu 24.04 and 26.04 and Debian 12 and 13 in a privileged
container with `cgroupns_mode: host`. Five things in the scenario exist only because
the target is a container, not a host:

- a bind mount of the container's nested `user.slice` over `/sys/fs/cgroup/user.slice`.
  Rootless `dockerd` hardcodes that path, but under `cgroupns_mode: host` systemd puts
  `user.slice` under `/sys/fs/cgroup/docker/<id>/`. Upstream issue
  [moby/moby#53531](https://github.com/moby/moby/issues/53531); the fix is unreleased.
- `storage-driver: vfs`. The container's `/home` is itself overlayfs, so a nested
  overlay driver (or `fuse-overlayfs`) fails at `docker run`.
- `docker_setuptool_args: [--skip-iptables]`. The setuptool probes `/lib/modules` for
  `nf_tables`, which a container does not have.
- `/sys/kernel/security` mounted on the Ubuntu platforms only, so `apparmor.service`
  clears its `AssertPathIsReadWrite` and the `userns` profile can load. Without it the
  AppArmor branch fails on the handler. Debian ships no `sysctl.d` file activating the
  restriction, so it never reaches that branch.
- the Debian containers clear `kernel.apparmor_restrict_unprivileged_userns`. The knob
  is not namespaced, so they read the runner's kernel, and GitHub's Ubuntu runners set
  it. On a real Debian host it is `0`; forcing it keeps the sysctl branch exercised
  rather than every platform landing on the AppArmor one.

None of these apply to a real host; they are not role defaults.

## Known limitations of rootless mode

- No `--privileged`, no AppArmor profiles on containers, no arbitrary `--cap-add`
  beyond what the user namespace governs.
- Resource limits need cgroup v2 and systemd.
- The `IPAddress` in `docker inspect` is inside the rootlesskit network namespace and
  is not reachable from the host without `nsenter`.
- `docker run -p` does not preserve the client source address by default. See
  `TODO.md` for the port-driver options and what each costs.
