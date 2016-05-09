# ipv6-scripts

These are Debian scripts to help people setup a dual-stack IPv4/IPv6 gateway on ISPs who hand out an IPv6 block via DHCP. They have only been tested with Comcast and Orange (France). Without minor modifications they can certainly work for other ISPs and Linux distributions.

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
* Configure isc-dhcp-client for your ISP (see below for Orange France)
 * /etc/dhcp/dhclient.conf for IPv4 connectivity
 * /etc/dhcp/dhclient6.conf for IPv6 connectivity

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

## DHCP configuration for Orange France

This configuration works for most FTTH and VDSL subscribers since first trimester of 2016.

Also read this very long and interesting forum topic (in French) : https://lafibre.info/remplacer-livebox/remplacer-la-livebox-sans-pppoe/

### IPv4

Sample config for /etc/dhcp/dhclient.conf :

```
option rfc3118-authentication code 90 = string;

#Replace eth0 with your external interface (VLAN must be 832 for Orange)
interface "eth0.832" {
	#If you prefer to use Google public DNS servers
        prepend domain-name-servers 8.8.8.8, 8.8.4.4;
        
        #Orange France specific options
        send vendor-class-identifier "sagem";
        send user-class "+FSVDSL_livebox.Internet.softathome.Livebox3";
        
        #Authentication to Orange France DHCP server
        #Replace "xx:xx:xx:xx:xx:xx:xx" with hex value of your Orange France login (the characters after "fti/")
        #Just fill this form : https://jsfiddle.net/kgersen/45zudr15/embedded/result/
        send rfc3118-authentication 00:00:00:00:00:00:00:00:00:00:00:66:74:69:2f:xx:xx:xx:xx:xx:xx:xx;
        
        request subnet-mask, routers,
                domain-name, broadcast-address, dhcp-lease-time,
                dhcp-renewal-time, dhcp-rebinding-time,
                rfc3118-authentication;
}
```

### IPv6

Sample config for /etc/dhcp/dhclient.conf :

```
option dhcp6.auth code 11 = string;
option dhcp6.vendorclass code 16 = string;
option dhcp6.userclass code 15 = string;

#Replace eth0 with your external interface (VLAN must be 832 for Orange)
interface "eth0.832" {
	#Orange France specific options
        send dhcp6.vendorclass  00:00:04:0e:00:05:73:61:67:65:6d;
        send dhcp6.userclass 00:2b:46:53:56:44:53:4c:5f:6c:69:76:65:62:6f:78:2e:49:6e:74:65:72:6e:65:74:2e:73:6f:66:74:61:74:68:6f:6d:65:2e:6c:69:76:65:62:6f:78:33;
        
        #Authentication to Orange France DHCP server
        #Replace "xx:xx:xx:xx:xx:xx:xx" with hex value of your Orange France login (the characters after "fti/")
        #Just fill this form : https://jsfiddle.net/kgersen/45zudr15/embedded/result/
        send dhcp6.auth 00:00:00:00:00:00:00:00:00:00:00:66:74:69:2f:xx:xx:xx:xx:xx:xx:xx;
        
        #Replace xx:xx:xx:xx:xx:xx with the MAC address of your external interface
        send dhcp6.client-id 00:01:00:01:1e:bf:f5:9d:xx:xx:xx:xx:xx:xx;
}
```

## Extra notes:

* You need to allow UDP traffic to/from fe80::/10 and to port 546/from port 547 - unless you have `nf_conntrack_dhcpv6` module available and use conntrack.
