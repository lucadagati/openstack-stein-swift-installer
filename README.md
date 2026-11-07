# OpenStack Stein + Swift — single-node installer

![OpenStack](docs/openstack-logo.png)

A Bash script that deploys a complete **OpenStack Stein** cloud on one Ubuntu 18.04 LTS machine: Keystone, Glance, Placement, Nova, Neutron, Horizon and **Swift** object storage (on a real unmounted disk or an automatically created virtual disk image). Intended for labs, teaching and reproducible test environments.

## Requirements

- Ubuntu 18.04 LTS (fresh install recommended), root access (`sudo`), Internet connectivity
- Optionally an **unmounted** disk/partition for Swift

## Install

```bash
git clone https://github.com/lucadagati/openstack-stein-swift-installer.git
cd openstack-stein-swift-installer
sudo ./Stein_Installation.sh            # Swift on a virtual disk image (hdd.img)
sudo ./Stein_Installation.sh /dev/sdX   # Swift on a dedicated disk
```

During the installation `apt` will ask about the `50-server.cnf` configuration file: answer **N** (keep the version installed by the script).

The script downloads the CirrOS test image and the MariaDB repository helper itself; neither is stored in this repository.

## What it does

1. Adds the host IP as `controller` in `/etc/hosts` and installs the Stein cloud archive packages.
2. Creates the databases and users for every service, then configures chrony, RabbitMQ and memcached.
3. Installs and configures Keystone → Glance → Placement → Nova → Neutron → Horizon → Swift, copying the templates in [conf_files/](conf_files/) and verifying each service (`token issue`, `image list`, `swift stat`, …).

## Docker

[docker/Dockerfile](docker/Dockerfile) builds an Ubuntu 18.04 systemd image with the packages pre-installed:

```bash
docker build -f docker/Dockerfile -t openstack .
docker run -dit --privileged --name openstack-stein-container --net=host \
  -v /sys/fs/cgroup:/sys/fs/cgroup:ro openstack
docker exec -it openstack-stein-container ./Stein_Installation.sh
```

## Repository layout

```
Stein_Installation.sh   installer
conf_files/             configuration templates for every service
admin-openrc            credentials for the admin user (sourced by the script)
demo-openrc             credentials for the demo user
docker/Dockerfile       containerised variant
```

## Security notice

All passwords (`ADMIN_PASS`, `MYUSER_PASS`, `*_DBPASS`, `RABBIT_PASS`, the MariaDB root `password`, …) are the **well-known defaults from the OpenStack install guide**. They are suitable only for isolated test setups. Change them consistently in the script, the `*-openrc` files and `conf_files/` before exposing the deployment.

OpenStack Stein and Ubuntu 18.04 are end-of-life; do not use this for production. The script was tested on a specific environment and may need adjustments elsewhere.
