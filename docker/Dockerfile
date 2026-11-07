# Build from the repository root:  docker build -f docker/Dockerfile -t openstack .
# Run:  docker run -dit --privileged --name openstack-stein-container --net=host \
#         -v /sys/fs/cgroup:/sys/fs/cgroup:ro openstack
# Then, inside the container:  ./Stein_Installation.sh

# Base image: Ubuntu 18.04
FROM ubuntu:18.04

ENV container docker
RUN apt-get update && apt-get install -y systemd && apt-get clean && rm -rf /var/lib/apt/lists/* \
    && rm -f /lib/systemd/system/multi-user.target.wants/* \
    && rm -f /etc/systemd/system/*.wants/* \
    && rm -f /lib/systemd/system/local-fs.target.wants/* \
    && rm -f /lib/systemd/system/sockets.target.wants/*udev* \
    && rm -f /lib/systemd/system/sockets.target.wants/*initctl* \
    && rm -f /lib/systemd/system/basic.target.wants/* \
    && rm -f /lib/systemd/system/anaconda.target.wants/*

RUN ln -sf /usr/share/zoneinfo/Europe/Rome /etc/localtime && echo "Europe/Rome" > /etc/timezone

# Update packages and install build dependencies
RUN apt-get update && apt-get upgrade -y && apt-get install -y \
    software-properties-common \
    curl \
    iproute2

# Add the OpenStack Stein cloud archive
RUN add-apt-repository cloud-archive:stein -y

RUN apt-get update && apt-get upgrade -y && apt-get install -y \
    sudo \
    net-tools \
    dialog \
    mariadb-server \
    iputils-ping \
    wget \
    vim  \
    git  \
    net-tools \
    chrony \
    python-pymysql \
    rabbitmq-server \
    memcached \
    python-memcache \
    keystone \
    glance \
    placement-api \
    swift \
    swift-proxy \
    python-swiftclient \
    python-keystoneclient \
    python-keystonemiddleware \
    xfsprogs \
    rsync \
    nova-api \
    nova-conductor \
    nova-consoleauth \
    nova-novncproxy \
    nova-scheduler \
    nova-compute \
    neutron-server \
    neutron-plugin-ml2 \
    neutron-linuxbridge-agent \
    neutron-l3-agent \
    neutron-dhcp-agent \
    neutron-metadata-agent \
    openstack-dashboard \
    python3-openstackclient
    
# Copy the installer from the local build context
COPY . /openstack-stein-swift-installer

# Working directory
WORKDIR /openstack-stein-swift-installer

# Make the installer executable
RUN chmod +x Stein_Installation.sh

# Boot systemd as init
#CMD cd /openstack-stein-swift-installer && /bin/bash -c "./Stein_Installation.sh"
CMD ["/sbin/init"]




