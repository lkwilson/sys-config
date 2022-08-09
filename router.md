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
[You can't really bridge with wifi](bridging.md)

## IPv6
I don't know how it works, so for this guide, I completely ignore it..

Some day.

# Networking
See [networking](networking.md) to setup the networking how you want it

# DNS and DHCP
See [DNS and DHCP](dns_and_dhcp.md) to setup dns and dhcp

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

## [IP Cheat sheet](ip_cheat_sheet.md)

# General Resources

A great resource for everything about networking
- http://intronetworks.cs.luc.edu

A great resource for securing your home server
- https://github.com/imthenachoman/How-To-Secure-A-Linux-Server
