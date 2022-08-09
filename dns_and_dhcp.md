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

# Disable `systemd-resolved`

If it's running, it's a local DNS service, so it'll conflict with `dnsmasq`.
```
systemctl disable systemd-resolved
```

# DNS

## root servers
```
sever=8.8.8.8
sever=8.8.4.4
```

## Security
Don't route DNS packets that root server's don't resolve anyways
```
domain-needed
bogus-priv
```

## cache
```
cache-size=1000
```

## Domain name
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

## Prevent adding `/etc/hosts` to dns
```
no-hosts
```

I haven't seen any issues with doing this.

## Override DNS names

### Block a lookup
```
address=/some-ad.com/
```

### Redirect a site to a local IP
```
address=/google.com/192.168.1.213
```

### Redirect your url to a local IP
```
address=/my-private-domain.com/192.168.1.213
```

# DHCP
## DHCP Settings
```
dhcp-range=192.168.0.50,192.168.0.150,255.255.255.0,12h
dhcp-option=option:router,192.168.0.1
dhcp-option=option:dns-server,192.168.0.1
```

## Make your DHCP faster but more bossy
```
dhcp-authoritative
```

## static ip via dhcp + mac
```
dhcp-host=aa:bb:cc:dd:ee:ff,192.168.1.151
```

## I guess you can add custom routes?
This routes 192.168.2.0/24 packets via the 192.168.0.1 router. This works if the
dnsserver is in a 192.168.1.0/24 network (I think).
```
dhcp-option=121,192.168.2.0/24,192.168.0.1
```

# General `dnsmasq` config

## Ignore system settings
Make the config file the only one
```
no-resolv
```

## restrict interfaces

This makes it run on all interfaces but ignore requests coming from interfaces
other than loopback and this one:
```
interface=eth0
```

### extra secure
If you super care about security, you can prevent the packet from reaching the
dnsmasq altogether with the following:
```
bind-interfaces
```

This binds the listener to the interface so that the packets from other
interfaces are instead blocked at the kernel level.

### specify lan instead of interface
This makes it run on all but respond on these lans.

```
listen-address=::1,127.0.0.1,192.168.1.1
```

# Extras
## PXE Server

If you want, it can be a pxe boot server as well.

# Sources

A great resource on dnsmasq
- https://wiki.archlinux.org/title/dnsmasq

