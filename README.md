# ipv6-scripts

These are Debian scripts to help people setup a v6 gateway on ISPs who hand out a v6 block. They have only been tested with Comcast and Orange (France). Without minor modifications they can certainly work for other ISPs and Linux distributions.

This repository is a fork of the [excellent work](https://github.com/jaymzh/v6-gw-scripts) done by [Phil Dibowitz](https://github.com/jaymzh). Thank you so much for these scripts and your extremely useful [blog post](https://www.phildev.net/phil/blog/?p=308).

## Setup

* Requirements : 
 * package "isc-dhcp-client" (at least version 4.1.1-P1-17)
 * package "isc-dhcp-server" (if you want to serve IPv4 addresses to your LAN, not covered by these scripts for now)
 * package "radvd"
 * package "vlan"
 * Kernel support for VLANs (8021q module) 

* Clone this repository on your server

* Create the symlinks :
 * /etc/network/if-up.d/99-ipv6 => 99-ipv6
 * /etc/network/if-up.d/nat => nat
 * /etc/dhcp/dhclient-exit-hooks.d/ipv6-setup => ipv6-setup
 * /etc/radvd.conf.tmpl => radvd.conf.tmpl

* Copy ipv6_prefix_dhclient.conf.example into /etc/ipv6_prefix_dhclient.conf and customize it for your environment

* Add entries to /etc/network/interfaces

## Example of /etc/network/interfaces

Do not reuse this "interfaces" configuration file but adapt it to your environment.
In this example :
* eth0 is the external interface
* eth0.832 (VLAN 832) is a specific VLAN that must be used with Orange France ISP
* eth1 is the internal interface

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
	address 192.168.1.100
        netmask 255.255.255.0
        broadcast 192.168.1.254

auto eth0.832
iface eth0.832 inet manual
	up dhclient -cf /etc/dhcp/dhclient.conf -v -pf /var/run/dhclient.vlan832.pid -lf /var/lib/dhcp/dhclient.vlan832.leases eth0.832
	post-down dhclient -x -pf /var/run/dhclient.vlan832.pid || true

iface eth0.832 inet6 manual

auto eth1
iface eth1 inet static
	address 192.168.0.100
	netmask	255.255.255.0
	broadcast 192.168.0.254

iface eth1 inet6 auto
```

## Extra notes:

* You need to allow UDP traffic to/from fe80::/10 and to port 546/from port 547 - unless you have `nf_conntrack_dhcpv6` module available and use conntrack.
