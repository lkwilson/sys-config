# router

Notes on how to turn a linux server into a common router.

OpenWRT is the best thing to use, but if you don't feel like installing that for
a playground, this is for you.

## Use Case

You have a computer with two network interfaces. One connected to a wan, the
other you would like a personal lan. As far as this page is concerned, it
doesn't matter whether the network interfaces are wifi or ethernet.

It only makes a difference in that, if one of them is a wifi interface, it
cannot be bridged. This doesn't consider the bridge case, so it doesn't affect
us here.

There's two types of wan. The internet and another router's lan. If your router
is directly hooked to the internet, then you'll want to make your router as
secure as possible. If your router is hooked to another router's lan, then you
don't have to be as careful.

My specific use case is that my wan interface is wifi, so wlan0. My lan
interface is ethernet, so eth0. My wan is connected to another router (my own),
so I favor being less secure.

# Configure interfaces

## A note on ifconfig/ifup/ifdown

Commands like ifconfig, ifup, ifdown, etc, are deprecated.

A lot of guides will have you edit `/etc/network/interfaces`, but it is an
ifconfig/ifup/ifdown thing, so it is deprecated.

Others will tell you iproute2 is the replacement, but it's not a network
manager.

Network managers seem to be `NetworkManager` or `systemd-networkd`.

If you want to use `/etc/network/interfaces`, you're more than welcome to. I ran
into an issue where, my eth0 got a static IP for ifup, and then dhcpd gave it a
169.254.x.x/16 address as well. I switched to `systemd-networkd`, and all was
well.

## I use `systemd-networkd`

For this, create two config files:
```
/etc/systemd/network/10-eth0.network
/etc/systemd/network/11-wlan0.network
```

I want my static lan, with dns, to be available and setup before my wan is setup
because I point my wan at my lan's dns.

## Lan Interface `/etc/systemd/network/10-eth0.network`
```
[Match]
Name=eth0

[Network]
Address=192.168.0.1/24
```

- `192.168.0.1` is the subnet I chose for my router

Do I need the following? Not sure yet.
```
DNS=192.168.0.1
```

## Wan Interface: `/etc/systemd/network/11-wlan0.network`
```
[Match]
Name=wlan0

[Network]
DHCP=yes
DNS=192.168.0.1
IgnoreCarrierLoss=3s
```

- `IgnoreCarrierLoss=3s` is a wifi thing. Move it to any interface files with wifi.

## dhcpcd service

This service will actually do the dhcp registration, so don't disable it. If we
were missing the config for the lan, then it would assign a 169.254.x.x/16
address for recovery.

## Sources:

- https://wiki.archlinux.org/title/systemd-networkd
- https://www.debian.org/doc/manuals/debian-reference/ch05.en.html
- https://eldon.me/arch-linux-home-router-systemd-configuration/

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

However, I have noticed that phones don't love the .lan suffix, so maybe stick
with the standard ones.

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

### Prevent adding /etc/hosts to dns

```
no-hosts
```

### Ignore system settings

Make the config file the only one

```
no-resolv
```

### PXE Server

If you want, it can be a pxe boot server as well.

## Sources

- https://wiki.archlinux.org/title/dnsmasq

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

# Routing

This is the complicated part because of security. DHCP and DNS only listen on
internal interfaces, but Routing obviously involves both.

## Tooling

`firewalld` and `iptables` seem to be the routing programs of choice.
`firewalld` is a wrapper around `iptables`, and has nice zone based firewall abstractions

## zones


### Create wan zone

```
firewall-cmd --permanent --new-zone=wan
```

### Add services if you want

You can probably live with only having access to the router from the lan, so
this isn't necessary.

```
firewall-cmd --permanent --zone=wan --add-service=ssh
```

### Set the target to DROP
The default response to connection attempts is to drop them and not respond.
Good for undesired connections, bad for icmp (see Unblocking ICMP types)

```
firewall-cmd --permanent --zone=wan --set-target=DROP
```

### Add masquerading (the nat)
```
firewall-cmd --permanent --zone=wan --add-masquerade 
```

### Add the wan interface to the zone:
```
firewall-cmd --permanent --zone=wan --change-interface=wlan0
```

### Unblocking ICMP types
target=DROP causes ICMP packets to also drop. This is bad because ICMP does
useful things that helps the router do better. For this reason, I unblock all of
them, but you can google the important ones and just enable those if you like.

ICMP types are a little wierd on firewalld. You have to block the ones you want,
and then invert the icmp block rules.

If we do nothing, by default, the ICMP type is blocked because of target, so we
have to do it this way:

```
firewall-cmd --permanent --zone=wan --add-icmp-block-inversion 
```

Now when we block an icmptype, it'll actually be unblocking.

```
firewall-cmd --get-icmptypes
```

This outputs all the icmp types available. Iterate over these and unblock them.
Here's some bash magic:

```
# store in bash array
icmptypes=($(firewall-cmd --get-icmptypes))
# expand array with prefix
firewall-cmd --permanent --zone=wan ${icmptypes[@]/#/--add-icmp-block=}
```

At this point, you can unblock things like ping, but some argue it's unecessary.
It's an interesting rabbit hole if you're curious.

### Create lan zone

You can create a custom lan, or you can just use the default. The percs of the
default are that new interfaces are automatically added. You should configure
that one anyways to lock it up or relax the security, so you might as well just
use that for the lan.

```
firewall-cmd --permanent --new-zone=lan
```

### A secure lan

You can make it very open or very closed.
```
firewall-cmd --permanent --zone=lan --set-target=DROP
firewall-cmd --permanent --zone=lan --add-service={ssh,dns,dhcp}
```

### An unsecured one

```
firewall-cmd --permanent --zone=lan --set-target=ACCEPT
```

### Other settings

You can add a source to whitelist a zone:
```
firewall-cmd --permanent --zone=secure_lan --add-source=1.1.1.1
```

You could add port forwarding if needed

## Potential requirements

I guess some machines may disable routing by default. I think the above
configurations will enable it (either dnsmasq or firewalld's masquerading).

Create a new file /etc/sysctl.d/ip_forward.conf and add the following:
```
net.ipv4.ip_forward=1
```

## Sources

- https://eldon.me/arch-linux-based-home-router-part-iii-firewalld-configuration/


# Resources

- http://intronetworks.cs.luc.edu
- https://github.com/imthenachoman/How-To-Secure-A-Linux-Server
