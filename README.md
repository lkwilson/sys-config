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

## General

### restrict interfaces

```
interface=eth0
```

This makes it run dhcp + dns on the eth0 interface.

### Ignore system settings

Make the config file the only one

```
no-resolv
```

# Routing

This is the complicated part because of security. DHCP and DNS only listen on
internal interfaces, but Routing obviously involves both.

## Tooling

`firewalld` and `iptables` seem to be the routing programs of choice.
`firewalld` is a wrapper around `iptables`, apparently.

