# router
Here's a reference guide on everything you need to turn your debian/linux box
into a router.

For serious routing, use OpenWRT.

For real security advice, I redirect you to the [github repo](https://github.com/imthenachoman/How-To-Secure-A-Linux-Server) cited at the bottom.

Feel free to make a pull request or open an issue.

## Motivation
### Problem
I want a private lan but only have public/shared wifi.

### Solution
This solution converts that wifi into an ethernet port with a private lan.

It's not a production solution, but it works pretty well for a few people's
worth of traffic.

## Network Layout
Suppose you have a network that routes to the internet. You can join this
network via one of your network interfaces (connect to wifi, plug into
ethernet). I call this the external/wan interface. This guide will refer to my wifi
interface, wlan0, as the external network interface.

You have another network interface that you want to be a LAN network. It will
have things like firewall, DHCP, DNS and NAT routing out the external interface.
I call this the internal/lan interface. This guide will refer to my ethernet
interface, eth0, as the internal network interface.

That means I convert a wifi network connected to internet into a LAN network
accesible via an ethernet port.

## My Lax Security Requirements
My use case has my router connected to another router's LAN, not directly to the
internet. The LAN is also owned by me but shared with trusted people, so I don't take
extreme measures with security. If you connect your router directly to the internet,
see external sources such as this [github repo](https://github.com/imthenachoman/How-To-Secure-A-Linux-Server).

## Bridging
My first thought was that if I sacrifice the private lan and just join the outer
lan, all I have to do is bridge my wifi interface with the ethernet one. Turns
out, that [you can't really brdige a wifi interface](https://serverfault.com/questions/152363/bridging-wlan0-to-eth0).
Mostly due to how wifi packets work and lack of router support.

That being said, if you have multiple ethernet ports on your linux box, you can
bridge them all together, and make the bridge interface the internal interface
for this guide. I haven't tested it, so it might not be that easy, but it seems
to make sense at least. There's then fancy vlan stuff you could probably do.
Create a DMZ for your IOT devices.

In my case, I just hook my external interface to a switch, and it works just
fine. If your external interface is a wifi access point, it'll probably also
work just fine.

### How to make a bridge
Use iproute2 tools such as `ip` and `bridge`:

```bash
ip link add name br0 type bridge
ip link set dev br0 up
ip link set dev eth0 master br0
ip link set dev eth1 master br0

# Undo
ip link set dev eth0 nomaster
ip link set dev eth1 nomaster
ip link del br0
```

## IPv6
I don't know how it works, so for this guide, I completely ignore it..

Some day.

# Configure IP Addresses
## A note on ifconfig/ifup/ifdown
Commands like ifconfig, ifup, ifdown, etc, are deprecated.

A lot of guides will have you edit `/etc/network/interfaces`, but it is an
ifconfig/ifup/ifdown thing, so while convenient, it's not recommended.

Others will tell you iproute2 is the replacement, but it's not a network
manager. Setting ip addresses with `ip addr` don't persist after reboot, and
adding ip commands to run level scripts seems like an advanced thing that could
easily be messed up.

Instead, we should use a network manager such as `NetworkManager` or `systemd-networkd`.

`systemd-networkd` is the one I picked for this guide.

### Notes
If you want to use `/etc/network/interfaces`, you're more than welcome to, but
here's issues I ran into.

#### eth0 had two IP addresses
My eth0 got a static IP from ifup, and then I assume `dhcpcd` gave it a
`169.254.x.x/16` address after dhcp failed.

I removed the `/etc/network/interfaces` config, and I switched to
`systemd-networkd`.  I assumed `dhcpcd` was also unecessary, so I disabled that
as well. However, eth0 didn't get any IP address at all at that point. I re
enabled `dhcpcd`, and it worked just as expected: one single static IP.

## Setup `systemd-networkd`
We use `systemd-networkd` as our network manager

Enable it:
```
systemctl enable systemd-networkd
```

Configure it by creating two config files:
```
/etc/systemd/network/10-eth0.network
/etc/systemd/network/11-wlan0.network
```

### Note on naming
I want my internal interface, eth0, to be available and setup before my external
interface, wlan0, is setup because I point my external interface at my lan's
dns.

I don't know if it actually matters, but it seems to work just fine this way.

## Lan Interface `/etc/systemd/network/10-eth0.network`
We configure the internal interface with a static IP and a subnet mask defining
its subnet

```
[Match]
Name=eth0

[Network]
Address=192.168.0.1/24
```

I will use `192.168.0.1/24` as my router's lan subnet.

The router will have a static IP of `192.168.0.1`. This can be set to whatever you want,
just make sure it matches the gateway in DHCP.

### DNS Note
The following might be useful / necessary, but it seems to work without it, so I
just leave it out.
```
DNS=192.168.0.1
```

### Gateway Note
Everything outside of this subnet range will be routed through the default
route, which is likely the external interface, since it's DHCP server will tell
it a gateway, which you'll notice we don't configure here. If we did, it would
likely break some things.

## Wan Interface: `/etc/systemd/network/11-wlan0.network`
```
[Match]
Name=wlan0

[Network]
DHCP=yes
DNS=192.168.0.1
IgnoreCarrierLoss=3s
```

I think `IgnoreCarrierLoss=3s` is a wifi thing. Move it to any interface files
with wifi. It doesn't necessarily go in the external interface configuration.

### DNS Note
We are hosting a DNS server on the internal interface, so we will use it as the
DNS server. That way, we don't have to trust the external network's DNS
settings, and we can use a DNS cache that's on the same machine. It might not be
necessary, but it's probably better.

## `dhcpcd` service

`systemd-networkd` configures the interface to use dhcp, `dhcpcd` actually does
the dhcp client work. For that reason, don't disable this service.

## Sources:

Great resources on systemd-networkd
- https://wiki.archlinux.org/title/systemd-networkd
- https://www.debian.org/doc/manuals/debian-reference/ch05.en.html

A great blog about more advanced config of systemd-networkd
- https://eldon.me/arch-linux-home-router-systemd-configuration/

# Router services
We want the linux box to have a DNS server and DHCP server.

While the DNS server isn't necessary, it's convenient for DNS name caching, and
for local dns name resolution.

`dnsmasq` seems to be the defacto. It will provide a dhcp and dns server all in
one. It's config file is here: `/etc/dnsmasq.conf`

```
systemctl enable dnsmasq
```

An alternative is `isc-dhcp-server`.

## Disable `systemd-resolved`

If it's running, it's a local DNS service, so it'll conflict with `dnsmasq`.
```
systemctl disable systemd-resolved
```

## DNS

### root servers
```
sever=8.8.8.8
sever=8.8.4.4
```

### Security
Don't route DNS packets that root server's don't resolve anyways
```
domain-needed
bogus-priv
```

### cache
```
cache-size=1000
```

### Domain name
You can set a domain name so DNS will resolve local network host dns names.

Recommended (but less standard):
```
local=/local/
domain=local
```

True standard:
```
local=/home.arpa/
domain=home.arpa
```

Non standard but nicest
```
local=/lan/
domain=lan
```

The problem with non standard is that a router could forward it to main servers
and leak network information, but the `local=` setting might fix that, so then
you can safely use the convenient .lan network.

However, you still have the problem of some devices not loving the .lan suffix,
so maybe stick with the standard ones. It may convert it into a google search.

### Prevent adding `/etc/hosts` to dns
```
no-hosts
```

I haven't seen any issues with doing this.

### Override DNS names

#### Block a lookup
```
address=/some-ad.com/
```

#### Redirect a site to a local IP
```
address=/google.com/192.168.1.213
```

#### Redirect your url to a local IP
```
address=/my-private-domain.com/192.168.1.213
```

## DHCP
### DHCP Settings
```
dhcp-range=192.168.0.50,192.168.0.150,255.255.255.0,12h
dhcp-option=option:router,192.168.0.1
dhcp-option=option:dns-server,192.168.0.1
```

### Make your DHCP faster but more bossy
```
dhcp-authoritative
```

### static ip via dhcp + mac
```
dhcp-host=aa:bb:cc:dd:ee:ff,192.168.1.151
```

### I guess you can add custom routes?
This routes 192.168.2.0/24 packets via the 192.168.0.1 router. This works if the
dnsserver is in a 192.168.1.0/24 network (I think).
```
dhcp-option=121,192.168.2.0/24,192.168.0.1
```

## General `dnsmasq` config

### Ignore system settings
Make the config file the only one
```
no-resolv
```

### restrict interfaces

This makes it run on all interfaces but ignore requests coming from interfaces
other than loopback and this one:
```
interface=eth0
```

#### extra secure
If you super care about security, you can prevent the packet from reaching the
dnsmasq altogether with the following:
```
bind-interfaces
```

This binds the listener to the interface so that the packets from other
interfaces are instead blocked at the kernel level.

#### specify lan instead of interface
This makes it run on all but respond on these lans.

```
listen-address=::1,127.0.0.1,192.168.1.1
```

## Extras
### PXE Server

If you want, it can be a pxe boot server as well.

## Sources

A great resource on dnsmasq
- https://wiki.archlinux.org/title/dnsmasq

# Routing

This is the complicated part because of security. DHCP and DNS only listen on
internal interfaces, but routing obviously involves both.

## Tooling

Routing seems to be done with the firewall. Or I guess more correctly, routing
just happens, and the firewall will manage *how* the routing actually happens.

`firewalld` (which wraps `iptables`) seems to be an easy-to-use, popular choice
of firewall.

## zones
We will create two firewall zones: wan and lan.
- wan will block everyting except icmp and it will be masquaraded (NAT).
- lan will be as open or closed as you like, but it will include DNS and DHCP services.

## Create wan zone
```
firewall-cmd --permanent --new-zone=wan
```

### Add masquerading (the NAT)
```
firewall-cmd --permanent --zone=wan --add-masquerade 
```
Note: This is what makes routing over a wifi interface work. Masquerading makes
the source address come from the router. Wifi can't always handle multiple
source addresses for a single client, which if bridged, the source address would
be all the clients in the lan.

### Set the target to DROP
The default response to connection attempts is to drop them and not respond.
Good for undesired connections, bad for icmp (see Unblocking ICMP types)

```
firewall-cmd --permanent --zone=wan --set-target=DROP
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

### Add services if you want
You can probably live with only having access to the router from the lan, so
this isn't necessary.

```
firewall-cmd --permanent --zone=wan --add-service=ssh
```

### Add the wan interface to the zone:
```
firewall-cmd --permanent --zone=wan --change-interface=wlan0
```

## Create lan zone
```
firewall-cmd --permanent --new-zone=lan
```

You can make it very open or very closed. If your router is hooked directly to
the internet, you probably want it to be more secure. If you have people on your
private lan that you don't trust, then make it more secure. If you don't care
and your wan is actually another trusted router's lan, then make this lan zone
less secure for convenience.

### Note about default zone
You can create a lan zone to use, but new network interfaces to the linux box
will automatically get added to the default zone. If this worries you, then
consider using the default zone (usually public) as the lan zone. Otherwise, you
can create a lan zone, and manually assign the network interface.

Since you likely have to secure the default zone anyways, it might be better to
just use the lan zone.

### A secure lan
```
firewall-cmd --permanent --zone=lan --set-target=DROP
# ICMP stuff (see above)
firewall-cmd --permanent --zone=lan --add-service={ssh,dns,dhcp}
```

### An unsecure lan
```
firewall-cmd --permanent --zone=lan --set-target=ACCEPT
```

### Other settings
You can add a source to whitelist a zone:
```
firewall-cmd --permanent --zone=secure_lan --add-source=1.1.1.1
```

You could add port forwarding if needed

## Sources

A great resource on firewalld
- https://wiki.archlinux.org/title/Firewalld

This blog does fancy things with vlans and DMZ zones
- https://eldon.me/arch-linux-based-home-router-part-iii-firewalld-configuration/

# Other things

## Manually enable routing packets

I guess some machines may disable routing by default. I think the above
configurations will enable it (either dnsmasq or firewalld's masquerading).

If not, you may have to do the following:
Create a new file /etc/sysctl.d/ip_forward.conf and add the following:
```
net.ipv4.ip_forward=1
```

I did not need to do this, and I did not test this.

## Sources

A great resource on firewalld
- https://wiki.archlinux.org/title/Firewalld

This blog does fancy things with vlans and DMZ zones
- https://eldon.me/arch-linux-based-home-router-part-iii-firewalld-configuration/

# General Resources

A great resource for everything about networking
- http://intronetworks.cs.luc.edu

A great resource for securing your home server
- https://github.com/imthenachoman/How-To-Secure-A-Linux-Server
