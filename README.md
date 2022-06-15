# router

Notes on how to turn a linux server into a common router.

OpenWRT is the best thing to use, but if you don't feel like installing that for
a playground, this is for you.

# DHCP + DNS

`dnsmasq` seems to be the defacto. It will provide a dhcp and dns server.


## config file

It's config file: `/etc/dnsmasq.conf`

## DNS

### root servers

```
#sever=/localnet/192.168.0.1
```
to
```
sever=8.8.8.8
sever=8.8.4.4
```

### Security

Don't route DNS packets that root server's don't resolve anyways

```
domain-needed
```

```
bogus-priv
```

### cache

```
#cache-size=150
```
to
```
cache-size=1000
```

### service name

```
sudo systemctl restart dnsmasq
```

### Disable systemd-resolved

I think it only runs on ubuntu, but if it's running, it's a local DNS service,
so it'll conflict dnsmasq.

### Domain name

Standard
```
local=/home.arpa/
domain=home.arpa
```
Less standard
```
local=/local/
domain=local
```
Non standard
```
local=/lan/
domain=lan
```

The problem with non standard is that a router could forward it to main servers
and leak network information, but the `local=` setting might fix that, so you
can then safely use the convenient .lan network.

### Override

Block a lookup
```
address=/some-ad.com/
```

Redirect a site to a local IP
```
address=/google.com/192.168.1.213
```

Redirect your url to a local IP
```
address=/my-private-domain.com/192.168.1.213
```

## DHCP

### DHCP Settings

```
dhcp-range=192.168.10.50,192.168.10.150,255.255.255.0,12h
dhcp-option=option:router,192.168.10.1
dhcp-option=option:dns-server,192.168.10.1
```

### Make your DHCP faster but more bossy

```
dhcp-authoritative
```

### I guess you can add custom routes?

This routes 192.168.2.0/24 packets via the 192.168.0.1 router. This works if the
dnsserver is in a 192.168.1.0/24 network (I think).

```
dhcp-option=121,192.168.2.0/24,192.168.0.1
```

### static ip via dhcp + mac

dhcp-host=aa:bb:cc:dd:ee:ff,192.168.1.151

## General

### restrict interfaces

This makes it run on all interfaces but ignore requests coming from interfaces other than loopback and this one:

```
interface=eth0
```

This binds the listener to the interface so that the packets from other
interfaces are instead blocked at the kernel level.

```
bind-interfaces
```

This makes it run on all but respond on these lans.

```
listen-address=::1,127.0.0.1,192.168.1.1
```

### Ignore system settings

Make the config file the only one

```
no-resolv
```

### PXE Server

If you want, it can be a pxe boot server as well.

## Sources

https://wiki.archlinux.org/title/dnsmasq

# Routing

This is the complicated part because of security. DHCP and DNS only listen on
internal interfaces, but Routing obviously involves both.

## Tooling

`firewalld` and `iptables` seem to be the routing programs of choice.
`firewalld` is a wrapper around `iptables`, apparently.


# Bridging

***Note! [Never Mind](https://serverfault.com/questions/152363/bridging-wlan0-to-eth0)***

***This doesn't actually work because WIFI isn't good at bridging with ethernet***

This is however useful for creating a bridge. The bridge can be what you run
dhcp, etc on, and then you have a router with many ethernet ports. You just add
them. Then you can probably do VLAN stuff, but I highly recommend somewhere else
for learning how to do that.

## Deprecated:

Suppose you have a wifi signal, a standard linux box, and a router. You could
turn the linux box into a router, or you could even install OpenWRT, and then
make the router a bridge network for WIFI. Or you could just bridge the linux
box and essentially use the linux box as a translator of wifi to ethernet:

Use iproute2 tools such as `ip` and `bridge`:

```bash
ip link add name br0 type bridge
ip link set dev br0 up
ip link set dev eth0 master br0
ip link set dev wlan0 master br0

# Undo
ip link set dev wlan0 nomaster
ip link set dev eth0 nomaster
ip link del br0
```
