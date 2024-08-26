+++
title = 'OpenWrt dnsmasq'
date = 2024-08-26T10:10:58+08:00
draft = false
tags = ['openwrt', 'dnsmasq']
+++


## What is Dnsmasq?

- Dnsmasq provides network infrastructure for small networks: DNS, DHCP, router advertisement and network boot. 
- It is designed to be lightweight and have a small footprint, suitable for resource constrained routers and firewalls. 
- It has also been widely used for tethering on smartphones and portable hotspots, and to support virtual networking in virtualisation frameworks. 
- Supported platforms include Linux (with glibc and uclibc), Android, *BSD, and Mac OS X. 
- Dnsmasq is included in most Linux distributions and the ports systems of FreeBSD, OpenBSD and NetBSD. 
- Dnsmasq provides full IPv6 support.


## How to set specific client gateway and dns server in Openwrt?

Client's mac address is `aa:bb:cc:dd:ee:ff`

Assuming that the client default gateway is 192.168.1.1, and the dns server is 192.168.1.1.

But you want to set the client gateway to 192.168.1.2 and dns server to 192.168.1.2.

### uci
```shell
uci set dhcp.@host[-1]=host
uci set dhcp.@host[-1].ip='192.168.1.100'
uci set dhcp.@host[-1].mac='aa:bb:cc:dd:ee:ff'
uci set dhcp.@host[-1].tag='bypass'

uci add dhcp tag
uci set dhcp.@tag[-1].name='bypass'
uci set dhcp.@tag[-1].dhcp_option='3,192.168.1.2 6,192.168.1.2'
uci set dhcp.@tag[-1].force='1'

uci commit dhcp

/etc/init.d/dnsmasq restart

```

### direct edit /etc/config/dhcp
```shell
config host
    option ip '192.168.1.100' # set client static ip
    option mac 'aa:bb:cc:dd:ee:ff'
    option tag 'bypass'

config tag 'bypass'
    option dhcp_option '3,192.168.1.2 6,192.168.1.2'
    option force '1'

/etc/init.d/dnsmasq restart
```

## Reference

- [Dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html)
- [OpenWrt](https://openwrt.org/docs/guide-user/base-system/dhcp)
