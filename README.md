# OpenStack Stein + Swift: single-node installer

![OpenStack](docs/openstack-logo.png)

A Bash script that deploys an **OpenStack Stein** cloud (Keystone, Glance, Placement, Nova, Neutron, Horizon and **Swift** object storage) on a single Ubuntu 18.04 LTS machine. Swift can use a real unmounted disk/partition or an automatically created virtual disk image. It is meant for labs, teaching and reproducible test environments.

## Table of contents

1. [What it is and why it exists](#1-what-it-is-and-why-it-exists)
2. [Architecture](#2-architecture)
3. [Requirements](#3-requirements)
4. [Installation](#4-installation)
5. [Configuration](#5-configuration)
6. [Usage and expected output](#6-usage-and-expected-output)
7. [Repository layout](#7-repository-layout)
8. [Testing and verification](#8-testing-and-verification)
9. [Troubleshooting](#9-troubleshooting)
10. [Limitations and security notes](#10-limitations-and-security-notes)
11. [Extending and customising](#11-extending-and-customising)
12. [Changes](#12-changes)

## 1. What it is and why it exists

Installing OpenStack by hand following the official install guide takes many manual steps. `Stein_Installation.sh` automates the "all-in-one" controller+compute variant of that guide, adds Swift (not part of the minimal guide) and ships pre-edited configuration templates so that the installation is repeatable. Swift configuration is wired to Keystone (`authtoken`/`keystoneauth` in the proxy pipeline).

## 2. Architecture

Everything runs on one host, which is both controller and compute node; its IP is registered as `controller` in `/etc/hosts`.

```
                       Ubuntu 18.04 host  ("controller")
 +--------------------------------------------------------------------+
 | MariaDB :3306   RabbitMQ (user openstack)   memcached :11211  chrony |
 |                                                                    |
 | Keystone :5000   Glance :9292   Placement :8778   Nova :8774/:6080 |
 | Neutron :9696 (ML2 + linuxbridge, L3, DHCP, metadata)              |
 | Horizon (Apache)                                                   |
 | Swift: proxy :8080, account :6202, container :6201, object :6200   |
 |        rsync; storage = /srv/node/<device> (disk or hdd.img)       |
 | nova-compute with libvirt virt_type=qemu (software emulation)      |
 +--------------------------------------------------------------------+
```

Installation order in the script: `host_config`, `install_stein_packages`, `configuring_db`, `chrony`, `rabbitmq`, `memcached`, `keystone`, `glance`, `placement`, `nova`, `neutron`, `horizon`, `swift_install`.

## 3. Requirements

- Ubuntu 18.04 LTS (the Stein cloud archive `cloud-archive:stein` targets Bionic), a fresh machine or VM is strongly recommended. The script modifies the host heavily.
- Root access (the script exits if not run as root), Internet access (apt, `cirros-0.4.0` image from download.cirros-cloud.net, MariaDB repository helper from downloads.mariadb.com).
- A default route: the host IP and NIC are derived from `ip route get 8.8.8.8`.
- Disk: about 3.6 GB extra for the default Swift virtual disk (`dd bs=3M count=1200`); alternatively a dedicated **unmounted** block device.
- RAM/CPU sizing is not specified by the repo (not verified); a typical all-in-one OpenStack lab needs several GB of RAM.
- Docker variant (optional): Docker able to run a `--privileged` systemd container with `--net=host`.

Not verified in this environment: a full run of the installer (it needs Ubuntu 18.04, modifies the system, and was deliberately not executed here).

## 4. Installation

```bash
git clone https://github.com/lucadagati/openstack-stein-swift-installer.git
cd openstack-stein-swift-installer
sudo ./Stein_Installation.sh            # Swift on a virtual disk image (~/hdd.img)
sudo ./Stein_Installation.sh /dev/sdX   # Swift on a dedicated, unmounted disk
```

Run it from the repository root: the script uses relative paths (`./conf_files/...`, `admin-openrc`). During the upgrade `apt` may ask what to do about the `50-server.cnf` configuration file: answer **N** (keep the installed version).

Optional: set a hostname resolution or static IP beforehand, as the IP is detected automatically.

### Docker variant

[docker/Dockerfile](docker/Dockerfile) builds an Ubuntu 18.04 image that boots systemd (`/sbin/init`) with all packages pre-installed and the repository copied to `/openstack-stein-swift-installer`:

```bash
docker build -f docker/Dockerfile -t openstack .
docker run -dit --privileged --name openstack-stein-container --net=host \
  -v /sys/fs/cgroup:/sys/fs/cgroup:ro openstack
docker exec -it openstack-stein-container ./Stein_Installation.sh
```

Caution: `--privileged --net=host` gives the container wide access to the host; use only on a disposable machine. Not verified in this environment.

## 5. Configuration

There is no config file for the installer; variables are defined near the end of `Stein_Installation.sh`, and several secrets are hard-coded in the function bodies and templates.

### Script variables

| Variable | Default | Meaning |
|---|---|---|
| `maria_db_root_password` | `password` | MariaDB root password set by the script |
| `maria_db_port` | `3306` | MariaDB port used by `maria_db_connect` |
| `keystone_db_*`, `glance_db_*`, `placement_db_*`, `nova_db_*`, `neutron_db_*` | names equal to the service; passwords `KEYSTONE_DBPASS`, `GLANCE_DBPASS`, `PLACEMENT_DBPASS`, `NOVA_DBPASS`, `NEUTRON_DBPASS` | Database names/users/passwords. Databases created: keystone, glance, placement, nova_api, nova, nova_cell0, neutron |
| `ip` | auto (`ip route get 8.8.8.8`, field 7) | Host IP written to `/etc/hosts` as `controller` and injected into nova, neutron, memcached, horizon, rsync and the Swift endpoints |
| `network_interface` | auto (field 5) | Provider NIC for `physical_interface_mappings = provider:<nic>` |
| `object_storage_disk` | first CLI argument, empty by default | Swift device. Empty: creates `~/hdd.img` of 3.6 GB (home of `$SUDO_USER`, or `/root`) |

### Hard-coded credentials (OpenStack install guide defaults)

| Item | Value |
|---|---|
| Keystone admin (`admin-openrc`) | user `admin`, password `ADMIN_PASS`, project `admin` |
| Demo user (`demo-openrc`) | user `myuser`, password `MYUSER_PASS`, project `myproject` |
| Service users | `glance` GLANCE_PASS, `placement` PLACEMENT_PASS, `nova` NOVA_PASS, `neutron` NEUTRON_PASS, `swift` SWIFT_PASS |
| RabbitMQ | user `openstack`, password `RABBIT_PASS` |
| Swift hashes (`conf_files/swift.conf`) | `HASH_PATH_SUFFIX`, `HASH_PATH_PREFIX` |
| Region / auth URL | `RegionOne`, `http://controller:5000/v3` |

To change a password, edit it consistently in the script, the `*-openrc` files and the matching `conf_files/*.conf` (for instance `keystone.conf` contains `connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone`).

### Templates in `conf_files/`

Copied to `/etc/...` by the script; some lines are rewritten with `sed` afterwards: `my_ip` and `novncproxy_base_url` (nova.conf), `physical_interface_mappings` and `local_ip` (linuxbridge_agent.ini), `nova_metadata_host` (metadata_agent.ini), `OPENSTACK_HOST` and cache `LOCATION` (local_settings.py), `address` (rsyncd.conf), `-l` (memcached.conf). Note that `nova-compute.conf` sets `virt_type=qemu` (no hardware acceleration needed).

## 6. Usage and expected output

After the run, load the credentials and use the CLI:

```bash
. admin-openrc
openstack token issue
openstack image list          # should list the image "cirros5" (CirrOS 0.4.0, public)
openstack compute service list
openstack network agent list
swift stat                    # account statistics
```

Horizon is served by Apache at `http://<host-ip>/horizon` (standard Ubuntu path; the exact URL depends on the `openstack-dashboard.conf` template, not checked live). Swift proxy endpoint: `http://<host-ip>:8080/v1/AUTH_<project_id>`.

The installer itself prints the verification commands as it goes (`token issue` for admin and myuser, `image list`, `placement-status upgrade check`, `nova-manage cell_v2 list_cells`, `openstack compute service list`, `openstack catalog list`, `nova-status upgrade check`, `openstack network agent list`, `swift stat`). Their output was not observed here.

The script does **not** create a provider/self-service network, flavors, keypairs or security-group rules; create them with the `openstack` CLI before booting instances.

## 7. Repository layout

```
Stein_Installation.sh   the installer (one function per service, run in sequence)
conf_files/             templates copied to /etc: keystone, glance (api, registry), placement, nova,
                        nova-compute, neutron (+ ml2, linuxbridge, l3, dhcp, metadata), horizon
                        (local_settings.py, openstack-dashboard.conf), MariaDB (50-server.cnf),
                        chrony, swift (proxy, account, container, object, swift.conf), rsyncd
admin-openrc            environment for the admin user (sourced by the script and by you)
demo-openrc             environment for the demo user
docker/Dockerfile       Ubuntu 18.04 + systemd image with packages pre-installed
docs/openstack-logo.png logo used in this README
.gitignore              hdd.img, *.img, mariadb_repo_setup, *.log
```

Artefacts created in the working directory during a run: `cirros-0.4.0-x86_64-disk.img`, `mariadb_repo_setup`, and the Swift `*.builder` / `*.ring.gz` files.

## 8. Testing and verification

On a target machine, after the script finishes:

1. `. admin-openrc && openstack token issue` returns a token.
2. `openstack service list` shows glance, placement, nova, neutron, swift (Keystone itself is bootstrapped).
3. `openstack image list` shows `cirros5`.
4. `openstack compute service list` and `openstack network agent list` show services/agents `up`.
5. `swift stat`, then `swift upload test <file>` and `swift list test`.
6. `findmnt /srv/node/<device>` shows the Swift device mounted.

Verified in this environment: `bash -n Stein_Installation.sh` passes; `shellcheck` (via the `shellcheck-py` package) reports a single warning (SC2034, `current_path` is unused), no errors. **Not verified in this environment:** the actual installation on Ubuntu 18.04 or in Docker, any service behaviour, Horizon access.

## 9. Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `This script must be run as root` | Use `sudo ./Stein_Installation.sh` |
| `./conf_files/...: No such file` or `admin-openrc: No such file` | Run from the repository root |
| `ip`/`network_interface` empty, bad `/etc/hosts` entry | No default route, or the `ip route get` output has no `src` as 7th field (host without a gateway). Set `ip` and `network_interface` manually in the script |
| `add-apt-repository cloud-archive:stein` fails | Not Ubuntu 18.04, or no network. Stein is end-of-life; the archive may be removed |
| `dpkg` prompts about `50-server.cnf` | Answer N |
| `mysql` access denied during DB creation | Root password already set to something other than `password`; edit `maria_db_root_password` or reset MariaDB on a fresh machine |
| Swift: `mount: ... failed` or `mkfs.xfs` error | Device in use/mounted, or `xfsprogs` missing; for the image, check free space (3.6 GB) and that loop devices are available (not in unprivileged containers) |
| Horizon not reachable | Check `systemctl status apache2` and `/etc/apache2/conf-available/openstack-dashboard.conf`; make sure the conf is enabled (`a2enconf openstack-dashboard`, not verified) |
| `openstack` commands time out | `controller` not resolving to the host IP, or services still starting; check `/etc/hosts` and `systemctl status` |
| Re-running the script fails | Database creation, users and endpoints are not idempotent. Reinstall on a clean machine |

## 10. Limitations and security notes

- All credentials are the well-known guide defaults (see section 5): use only on isolated test setups. MariaDB (`bind-address = 0.0.0.0`) and the Swift services (`bind_ip = 0.0.0.0` in the account server template) listen on all interfaces; RabbitMQ has the management plugin enabled with an administrator user. Firewall the host.
- The installer replaces the distribution's MariaDB with 10.5 from downloads.mariadb.com (inside the `neutron` function: `apt-get remove mariadb-server`, repo setup script downloaded and executed as root, reinstall). This runs remote code from the Internet as root; review it before use.
- `nobarrier` is set on the Swift mount: acceptable for a lab, unsafe for data you care about.
- OpenStack Stein and Ubuntu 18.04 are end-of-life: no security updates. Not for production.
- Single node, no HA, no Cinder, no tenant networks created. Compute uses QEMU emulation (slow).
- The script is not idempotent and has no error handling (`set -e` is not used); a failing step does not stop the run.
- The upstream author tested on a specific environment only; Ubuntu 20.04 was not verified.

## 11. Extending and customising

- Change passwords, DB names or the DB root password: variables at the end of the script plus the matching templates (section 5).
- Use another disk: pass its path as the first argument.
- Add a service (for example Cinder): add a function modelled on the existing ones (copy template, create user/service/endpoints, sync DB, restart) and call it in the sequence at the bottom of the script.
- Use real virtualisation: edit `conf_files/nova-compute.conf` (`virt_type=kvm`) if the CPU exposes `vmx`/`svm` (the script prints `egrep -c '(vmx|svm)' /proc/cpuinfo`).
- Restrict network exposure: change `bind-address` in `conf_files/50-server.cnf` and `bind_ip` in the Swift templates.

## 12. Changes

- `Stein_Installation.sh`: `object_storage_disk=${1:-"hdd.img"}` became `${1:-}`. With the old default the variable was never empty, so the "use a default virtual disk in the user's home" branch of `swift_install()` was dead code; the image was created at a relative path (`./hdd.img`) and that relative path was written to `/etc/fstab`, which breaks the mount on reboot or when run from another directory. Now, without argument, the image is created as `~/hdd.img` (absolute path) as the code's comments intended. Checked with `bash -n` and `shellcheck` only; not executed.
- README rewritten as a full guide.
