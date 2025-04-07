# Raspberry Pi2 / Pi3 OpenWrt Configuration Guide
A setup guide to transform a Raspberry Pi 2/3 into a Tailscale-connected OpenWrt router using Docker. This solution enables other devices to join the Tailscale mesh network without manually installing or configuring Tailscale, offering an easy way to extend secure, seamless connectivity across devices.<br>

Note: Port can be change at `/etc/config/uhttpd`

## Resize Partition
Install necessary tools:
```sh
opkg update && opkg install cfdisk resize2fs tune2fs fdisk
```
Check partitions:
```sh
fdisk -l
```
Resize using `cfdisk`:
```sh
cfdisk /dev/mmcblk0  # or cfdisk /dev/sda
```
Choose the second partition, resize, write changes, and quit.

Reboot the system:
```sh
reboot
```
Remount filesystem:
```sh
mount -o remount,ro /
```

Delete this partition:
```sh
tune2fs -O^resize_inode /dev/mmcblk0p2
fsck.ext4 /dev/mmcblk0p2 
```
Reboot the system:
```sh
reboot
```
resize filesystem:
```sh
resize2fs /dev/mmcblk0p2 # or #### resize2fs /dev/sda2
```
Verify changes:
```sh
df -h
```

---

## Install Additional Packages
```sh
opkg update && opkg install kmod-rt2800-lib kmod-rt2800-usb kmod-rt2x00-lib kmod-rt2x00-usb kmod-usb-core \
kmod-usb-uhci kmod-usb-ohci kmod-usb2 usbutils openvpn-openssl luci-app-openvpn nano kmod-usb-net-rtl8152
```

## Install Docker and its dependecies
```sh
opkg update && opkg install iptables-nft kmod-ipt-conntrack kmod-ipt-conntrack-extra kmod-ipt-conntrack-label kmod-nft-nat kmod-ipt-nat dockerd docker kmod-veth uxc procd-ujail # procd-ujail-console
```
## Install GUI: 
```sh
opkg update && opkg install luci-app-dockerman git git-http luci-app-statistics collectd-mod-thermal
```

## (Optional) memory fix and share root directory:
```sh
# nano /boot/cmdline.txt
# cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1 swapaccount=1
# nano /etc/rc.local (before exit 0 add below line)
# mount --make-shared /
```
## (Optional) Additional Packages:
```sh
opkg update && opkg install htop ncdu curl wget lsof blkid sudo 
```
## Reboot
```sh
reboot
```

# Download and flash using Raspberry PI Imager Tool:
Version for Openwrt is <a href="https://downloads.openwrt.org/releases/24.10.0/targets/bcm27xx/bcm2710/openwrt-24.10.0-bcm27xx-bcm2710-rpi-3-ext4-factory.img.gz">`openwrt-24.10.0-bcm27xx-bcm2710-rpi-3-ext4-factory`<a><br><br>

Note: Replace `network` and `firewall` file at /etc/config<br>

## network file here:
```bash
config interface 'loopback'
        option device 'lo'
        option proto 'static'
        option ipaddr '127.0.0.1'
        option netmask '255.0.0.0'

config globals 'globals'
        option ula_prefix 'fd94:282c:e352::/48'
        option packet_steering '1'

config device
        option name 'br-lan'
        option type 'bridge'
        list ports 'eth1'

config interface 'lan'
        option device 'br-lan'
        option proto 'static'
        option ipaddr '10.10.10.1'
        option netmask '255.255.255.0'
        option ip6assign '60'

config interface 'vpnclient'
        option proto 'none'
        option device 'tun0'

config interface 'wan'
        option proto 'dhcp'
        option device 'eth0'
        list dns '8.8.8.8'

config interface 'docker'
        option device 'docker0'
        option proto 'none'
        option auto '0'

config device
        option type 'bridge'
        option name 'docker0'

config interface 'tailscale'
        option ifname 'tailscale0'
        option proto 'none'
```

## firewall file here:
```bash
config defaults
        option syn_flood '1'
        option input 'REJECT'
        option output 'ACCEPT'
        option forward 'ACCEPT'

config zone
        option name 'lan'
        list network 'lan'
        option input 'ACCEPT'
        option output 'ACCEPT'
        option forward 'ACCEPT'

config zone
        option name 'wan'
        list network 'wan'
        list network 'wan6'
        option input 'ACCEPT'
        option output 'ACCEPT'
        option forward 'REJECT'
        option masq '1'
        option mtu_fix '1'

config forwarding
        option src 'lan'
        option dest 'wan'

config rule
        option name 'Allow-DHCP-Renew'
        option src 'wan'
        option proto 'udp'
        option dest_port '68'
        option target 'ACCEPT'
        option family 'ipv4'

config rule
        option name 'Allow-Ping'
        option src 'wan'
        option proto 'icmp'
        option icmp_type 'echo-request'
        option family 'ipv4'
        option target 'ACCEPT'

config rule
        option name 'Allow-IGMP'
        option src 'wan'
        option proto 'igmp'
        option family 'ipv4'
        option target 'ACCEPT'

config rule
        option name 'Allow-DHCPv6'
        option src 'wan'
        option proto 'udp'
        option dest_port '546'
        option family 'ipv6'
        option target 'ACCEPT'

config rule
        option name 'Allow-MLD'
        option src 'wan'
        option proto 'icmp'
        option src_ip 'fe80::/10'
        list icmp_type '130/0'
        list icmp_type '131/0'
        list icmp_type '132/0'
        list icmp_type '143/0'
        option family 'ipv6'
        option target 'ACCEPT'

config rule
        option name 'Allow-ICMPv6-Input'
        option src 'wan'
        option proto 'icmp'
        list icmp_type 'echo-request'
        list icmp_type 'echo-reply'
        list icmp_type 'destination-unreachable'
        list icmp_type 'packet-too-big'
        list icmp_type 'time-exceeded'
        list icmp_type 'bad-header'
        list icmp_type 'unknown-header-type'
        list icmp_type 'router-solicitation'
        list icmp_type 'neighbour-solicitation'
        list icmp_type 'router-advertisement'
        list icmp_type 'neighbour-advertisement'
        option limit '1000/sec'
        option family 'ipv6'
        option target 'ACCEPT'

config rule
        option name 'Allow-ICMPv6-Forward'
        option src 'wan'
        option dest '*'
        option proto 'icmp'
        list icmp_type 'echo-request'
        list icmp_type 'echo-reply'
        list icmp_type 'destination-unreachable'
        list icmp_type 'packet-too-big'
        list icmp_type 'time-exceeded'
        list icmp_type 'bad-header'
        list icmp_type 'unknown-header-type'
        option limit '1000/sec'
        option family 'ipv6'
        option target 'ACCEPT'

config rule
        option name 'Allow-IPSec-ESP'
        option src 'wan'
        option dest 'lan'
        option proto 'esp'
        option target 'ACCEPT'

config rule
        option name 'Allow-ISAKMP'
        option src 'wan'
        option dest 'lan'
        option dest_port '500'
        option proto 'udp'
        option target 'ACCEPT'

config zone 'docker'
        option input 'ACCEPT'
        option output 'ACCEPT'
        option forward 'ACCEPT'
        option name 'docker'
        list network 'docker'
```

## docker-compose for File Browser

Minimal `docker-compose.yml` may look like this:

```yaml
services:
  filebrowser:
    image: hurlenko/filebrowser
    user: "0:0"
    ports:
      - 8080:8080
    volumes:
      - ./root/docker:/data
      - ./config:/config
    environment:
      - FB_BASEURL=/filebrowser
    restart: always
```

## docker-compose for Tailscale

Minimal `docker-compose.yml` may look like this:

```yaml
services:
  tailscale:
    image: tailscale/tailscale:latest
    hostname: ts-host
    # Change Host Name Above
    container_name: tailscale
    environment:
      - TS_AUTHKEY=tailscale-auth-key
      # Change Auth Key Above
      - TS_EXTRA_ARGS=--accept-routes
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_ROUTES=192.168.1.0/24
      - TS_USERSPACE=false
      - TS_ACCEPT_DNS=true
    volumes:
      - ./ts:/var/lib
    devices:
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
      - NET_RAW
    restart: unless-stopped
    network_mode: host
```




# Allow All LAN Clients to Access Tailscale Network on OpenWrt (optional / Already Done) 

This guide explains how to allow all LAN devices on OpenWrt to access the Tailscale network.

## Scenario Summary
- OpenWrt is connected to Tailscale (running inside Docker).
- Devices on LAN (`192.168.1.0/24`) should be able to access Tailscale network routes (e.g., `100.x.x.x`, `192.168.x.0/24`).

---

## Step 1: Expose tailscale0 to OpenWrt system
Add this to OpenWrt network config:
```bash
uci set network.tailscale=interface
uci set network.tailscale.ifname='tailscale0'
uci set network.tailscale.proto='none'
uci commit network
/etc/init.d/network reload

```

---

## Step 2: Create Firewall Zone for Tailscale
```bash
uci add firewall zone
uci set firewall.@zone[-1].name='tailscale'
uci set firewall.@zone[-1].input='ACCEPT'
uci set firewall.@zone[-1].output='ACCEPT'
uci set firewall.@zone[-1].forward='ACCEPT'
uci set firewall.@zone[-1].network='tailscale'
uci set firewall.@zone[-1].masq='1'    # NAT is essential here!
uci commit firewall
```

---

## Step 3: Allow Forwarding from LAN to Tailscale
```bash
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='lan'
uci set firewall.@forwarding[-1].dest='tailscale'
uci commit firewall
/etc/init.d/firewall restart
```

### (Optional) Allow DNS too (if needed)
If your LAN clients need to resolve .ts.net domains:
```bash
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-DNS-to-Tailscale'
uci set firewall.@rule[-1].src='lan'
uci set firewall.@rule[-1].dest='tailscale'
uci set firewall.@rule[-1].proto='udp'
uci set firewall.@rule[-1].dest_port='53'
uci set firewall.@rule[-1].target='ACCEPT'
uci commit firewall
/etc/init.d/firewall restart
```

---

## (optional) Step 4: Commit and Restart Firewall
```bash
uci commit firewall
/etc/init.d/firewall restart
```

---

## Outcome
All LAN devices connected to OpenWrt can now access the Tailscale network and any routes advertised within it.

