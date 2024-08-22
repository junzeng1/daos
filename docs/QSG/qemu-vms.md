# Set Up DAOS Using QEMU

## Goal

This document gives the basic steps to set up an experimental DAOS system using QEMU VMs. This system has the minimal configuration of two VMs. One VM acts as the DAOS server and admin. The other one acts as the DAOS client. You can create pools, containers, and access containers through FUSE. You can experiment with DAOS even if you don't have no spare NVMe disk.

## Prerequisites

1.  A modern laptop with CPU virtualization support
    - Check your CPU by running `cat /proc/cpuinfo` and finding the `vmx` `sse4_2` flags. Your CPU should have at least 4 cores.
2.  16GB RAM or more
3.  QEMU
    - It is suggested to use a Linux based operating system. In principle you can also use other OSes but how to setup network is quite different from this document.
4.  libvirt and QEMU driver
5.  dnsmasq
6.  Rocky Linux 8.9 minimal ISO
7.  Root privilege on the host machine

## Steps

Assume you have both hardware and software prepared.

1.  Prepare disks for QEMU VMs.
```
# the disk image for daos-server
qemu-img create -f qcow2 daos-server.qcow2 40G
# the disk image for the emulated nvme disk for DAOS Tier 1
qemu-img create -f qcow2 qemu-nvm-disk1.qcow2 16G
# the disk image for daos-client
qemu-img create -f qcow2 daos-client.qcow2 20G
```
2.  Set up the network.
We use tap networks to connect the VMs. To automatically set up the guest network this can be done with `/etc/qemu-ifup`. This script will be called by QEMU. Put these contents in this script, more info on [this page](https://wiki.qemu.org/Documentation/Networking/NAT). It basically creates a network bridge if non-existent, and runs a dnsmasq service on this bridge to assign IP addresses and act as a gateway. Make sure that there is no other dnsmasq running as it may conflict with our dnsmasq. If you have virt-manager installed, make sure dnsmasq processes started by virt-manager are closed first.
```
#!/bin/sh
#
# Copyright IBM, Corp. 2010  
#
# Authors:
#  Anthony Liguori <aliguori@us.ibm.com>
#
# This work is licensed under the terms of the GNU GPL, version 2.  See
# the COPYING file in the top-level directory.

# Set to the name of your bridge
BRIDGE=br0

# Network information
NETWORK=192.168.53.0
NETMASK=255.255.255.0
GATEWAY=192.168.53.1
DHCPRANGE=192.168.53.2,192.168.53.254

# Optionally parameters to enable PXE support
TFTPROOT=
BOOTP=

do_brctl() {
    brctl "$@"
}

do_ifconfig() {
    ifconfig "$@"
}

do_dd() {
    dd "$@"
}

do_iptables_restore() {
    iptables-restore "$@"
}

do_dnsmasq() {
    dnsmasq "$@"
}

check_bridge() {
    if do_brctl show | grep "^$1" > /dev/null 2> /dev/null; then
    return 1
    else
    return 0
    fi
}

create_bridge() {
    do_brctl addbr "$1"
    do_brctl stp "$1" off
    do_brctl setfd "$1" 0
    do_ifconfig "$1" "$GATEWAY" netmask "$NETMASK" up
}

enable_ip_forward() {
    echo 1 | do_dd of=/proc/sys/net/ipv4/ip_forward > /dev/null
}

add_filter_rules() {
do_iptables_restore <<EOF
# Generated by iptables-save v1.3.6 on Fri Aug 24 15:20:25 2007
*nat
:PREROUTING ACCEPT [61:9671]
:POSTROUTING ACCEPT [121:7499]
:OUTPUT ACCEPT [132:8691]
-A POSTROUTING -s $NETWORK/$NETMASK -j MASQUERADE 
COMMIT
# Completed on Fri Aug 24 15:20:25 2007
# Generated by iptables-save v1.3.6 on Fri Aug 24 15:20:25 2007
*filter
:INPUT ACCEPT [1453:976046]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [1605:194911]
-A INPUT -i $BRIDGE -p tcp -m tcp --dport 67 -j ACCEPT 
-A INPUT -i $BRIDGE -p udp -m udp --dport 67 -j ACCEPT 
-A INPUT -i $BRIDGE -p tcp -m tcp --dport 53 -j ACCEPT 
-A INPUT -i $BRIDGE -p udp -m udp --dport 53 -j ACCEPT 
-A FORWARD -i $1 -o $1 -j ACCEPT 
-A FORWARD -s $NETWORK/$NETMASK -i $BRIDGE -j ACCEPT 
-A FORWARD -d $NETWORK/$NETMASK -o $BRIDGE -m state --state RELATED,ESTABLISHED -j ACCEPT 
-A FORWARD -o $BRIDGE -j REJECT --reject-with icmp-port-unreachable 
-A FORWARD -i $BRIDGE -j REJECT --reject-with icmp-port-unreachable 
COMMIT
# Completed on Fri Aug 24 15:20:25 2007
EOF
}

start_dnsmasq() {
    do_dnsmasq \
    --strict-order \
    --except-interface=lo \
    --interface=$BRIDGE \
    --listen-address=$GATEWAY \
    --bind-interfaces \
    --dhcp-range=$DHCPRANGE \
    --conf-file="" \
    --pid-file=/var/run/qemu-dnsmasq-$BRIDGE.pid \
    --dhcp-leasefile=/var/run/qemu-dnsmasq-$BRIDGE.leases \
    --dhcp-no-override \
    ${TFTPROOT:+"--enable-tftp"} \
    ${TFTPROOT:+"--tftp-root=$TFTPROOT"} \
    ${BOOTP:+"--dhcp-boot=$BOOTP"}
}

setup_bridge_nat() {
    if check_bridge "$1" ; then
    create_bridge "$1"
    enable_ip_forward
    add_filter_rules "$1"
    start_dnsmasq "$1"
    fi
}

setup_bridge_vlan() {
    if check_bridge "$1" ; then
    create_bridge "$1"
    start_dnsmasq "$1"
    fi
}

setup_bridge_nat "$BRIDGE"

if test "$1" ; then
    do_ifconfig "$1" 0.0.0.0 up
    do_brctl addif "$BRIDGE" "$1"
fi
```
Then add a sudo rule for qemu-system-x86\_64 in *etc/sudoers.d*.
```
<username> blackbox=(root) NOPASSWD: /usr/bin/qemu-system-x86_64
```
3.  Install guest operating systems.

Run these two commands. Select "boot from cdrom". Follow the installation guide to install the OSes. I use Rocky Linux 8.9 in this guide.
```
sudo qemu-system-x86_64 -M q35,accel=kvm,kernel-irqchip=split -cpu host -smp 3 -m 12288 -device intel-iommu,intremap=on -drive file=<image-dir>/daos-server.qcow2,if=virtio -device virtio-net,netdev=mynet0,mac=52:54:00:12:34:56 -netdev tap,id=mynet0 -drive file=<image-dir>/qemu-nvm-disk1.qcow2,if=none,id=nvm1 -device nvme,serial=deadbeef,drive=nvm1 -cdrom <iso-dir>/Rocky-8.9-x86_64-minimal.iso -boot menu=on &
sudo qemu-system-x86_64  -M q35,accel=kvm,kernel-irqchip=split -cpu host -smp 1 -m 2048 -device intel-iommu,intremap=on -drive file=<image-dir>/daos-client.qcow2,if=virtio -device virtio-net,netdev=mynet1,mac=52:54:00:12:34:57 -netdev tap,id=mynet1 -cdrom <iso-dir>/Rocky-8.9-x86_64-minimal.iso -boot menu=on
```
After OS installation is finished. Remove the `-cdrom` and `-boot` options from the QEMU commands to boot VMs normally. Next is to install DAOS software.
```
sudo qemu-system-x86_64 -M q35,accel=kvm,kernel-irqchip=split -cpu host -smp 3 -m 12288 -device intel-iommu,intremap=on -drive file=<image-dir>/daos-server.qcow2,if=virtio -device virtio-net,netdev=mynet0,mac=52:54:00:12:34:56 -netdev tap,id=mynet0 -drive file=<image-dir>/qemu-nvm-disk1.qcow2,if=none,id=nvm1 -device nvme,serial=deadbeef,drive=nvm1 &
sudo qemu-system-x86_64  -M q35,accel=kvm,kernel-irqchip=split -cpu host -smp 1 -m 2048 -device intel-iommu,intremap=on -drive file=<image-dir>/daos-client.qcow2,if=virtio -device virtio-net,netdev=mynet1,mac=52:54:00:12:34:57 -netdev tap,id=mynet1 
```
By default the network won't automatically connect in Rocky Linux 8.9, and ssh service is also not enabled as well. Run these commands to enable autoconnect behavior and sshd service. You can use ssh to connect to your VMs next time.
```
# enable the default connection
nmcli c up enp0s2
# enable autoconnect for this connection
nmcli c modify enp0s2 connection.autoconnect "yes"
# check if connection.autoconnect is enabled
nmcli c c show enp0s2
# enable sshd service
systemctl enable sshd.service
# start sshd service
systemctl start sshd.service
```

4.  Install DAOS software.

I follow these [steps](https://docs.daos.io/latest/QSG/setup_rhel/) to install both the DAOS server, DAOS admin, and DAOS client. I install daos-server and daos-admin on the first VM, and install daos-client on the second VM. You can follow the steps before Step "Create Configuration Files" to install DAOS software. Then come back to update config files.

5.  Update config files.

Update the daos-server config file `/etc/daos/daos_server.yml` on daos-server. You may need to update "access\_points", "fabric\_iface" and "bdev\_list". Update "access\_points" accordingly if you name daos-server differently. Check if the network device has the same name as listed under "fabric\_iface". Look in the output of `lspci` for "bdev\_list". The info for our NVMe controller is like *??:??:? Non-Volatile memory controller: Red Hat, Inc. QEMU NVM Express Controller (rev 02)*. Prefix *??:??.?* is the address of the NVMe devices.
```
name: daos_server
access_points:
- daos-server
port: 10001

nr_hugepages: 1024
system_ram_reserved: 4
disable_vfio: true 

transport_config:
    allow_insecure: false
    client_cert_dir: /etc/daos/certs/clients
    ca_cert: /etc/daos/certs/daosCA.crt
    cert: /etc/daos/certs/server.crt
    key: /etc/daos/certs/server.key
# Haven't got ofi+tcp to work with QEMU. Need further investigation.
provider: ofi+sockets
control_log_mask: DEBUG
control_log_file: /tmp/daos_server.log
helper_log_file: /tmp/daos_server_helper.log
engines:
-
    targets: 1
    pinned_numa_node: 0
    nr_xs_helpers: 0
    fabric_iface: enp0s2
    fabric_iface_port: 31316
    log_mask: DEBUG
    log_file: /tmp/daos_engine_0.log
    storage:
    -
        class: ram
        scm_mount: /mnt/daos0
        scm_size: 4
    -
        class: nvme
        bdev_list:
        - "0000:00:03.0"
```
Because we disable VFIO and use UIO, we have to run daos\_server.service as root. Open `/usr/lib/systemd/system/daos_server.service`, then change User and Group to root. Then run `systemctl daemon-reload` to reload systemd scripts.
```
...
[Service]
Type=simple
User=root
Group=root
...
```
Add the following line to `/etc/sysctl.conf` to preallocate huagepages. You need to reboot daos-server after updating `/etc/sysctl.conf`.
```
vm.nr_hugepages = 1024
```
Update the control config file `/etc/daos/daos_control.yml` on daos-server. You may need to update "hostlist" if the hostname of daos-server is different.
```
name: daos_server
port: 10001
hostlist:
- daos-server

transport_config:
    allow_insecure: false
    ca_cert: /etc/daos/certs/daosCA.crt
    cert: /etc/daos/certs/admin.crt
    key: /etc/daos/certs/admin.key  
```
Then update the agent config file `/etc/daos/daos_agent.yml` on daos-client. You may need to update "access\_points" if the hostname of daos-server is different.
```
name: daos_server
access_points:
- daos-server

port: 10001

transport_config:
    allow_insecure: false
    ca_cert: /etc/daos/certs/daosCA.crt
    cert: /etc/daos/certs/agent.crt
    key: /etc/daos/certs/agent.key
log_file: /tmp/daos_agent.log
control_log_mask: DEBUG
```
6.  Start services.
    On daos-server VM run `systemctl enable daos_server` to enable daos\_server service. Run `systemctl start daos_server` to start it. On daos-client VM run `systemctl enable daos_agent` to enable daos\_agent service. Run `systemctl start daos_agent` to start it. If everything is correct, you should be able to experiment with this minimal DAOS setup. Try to create pools, containers, and mount POSIX FSes.

## Possible Issues

1. If `dmg` returns error complaining not knowing daos-server. You can add the daos-server IP and name to /etc/hosts.
2. daos-server fails. Check the `/tmp/daos_server.log` for ERROR messages.
3. daos-server is up, but connections are rejected. Check if you have firewalld still turned on. Turn off firewalld `systemctl stop firewalld; systemctl disable firewalld`.