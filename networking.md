# Configure IP Addresses

`netplan` lets you configure network and lets you connect to wifi, so that's my
first pick. However, arch doesn't really support it, and raspbian doesn't have
it installed by default. My next pick is `systemd-networkd`, but you can't
configure wifi. To configure wifi, I like `iwd`. However, raspbian already has
`wpa_supplicant`, so for raspbian, I use that.

So there are 3 major ways to configure your network
- `netplan` (ubuntu)
- `systemd-networkd` + `iwd` (arch)
- `systemd-networkd` + `wpa_supplicant` (raspbian)

I'm sure `NetworkManager` works great for all cases, but it seemed more targeted
towards GUIs.

# A note on ifconfig/ifup/ifdown
Commands like ifconfig, ifup, ifdown, etc, are deprecated.

A lot of guides will have you edit `/etc/network/interfaces`, but it is an
ifconfig/ifup/ifdown thing, so while convenient, it's not recommended.

Others will tell you iproute2 is the replacement, but it's not a network
manager. Setting ip addresses with `ip addr` don't persist after reboot, and
adding ip commands to run level scripts seems like an advanced thing that could
easily be messed up.

## Notes
If you want to use `/etc/network/interfaces`, you're more than welcome to, but
here's issues I ran into.

### eth0 had two IP addresses
My eth0 got a static IP from ifup, and then I assume `dhcpcd` gave it a
`169.254.x.x/16` address after dhcp failed.

I removed the `/etc/network/interfaces` config, and I switched to
`systemd-networkd`.  I assumed `dhcpcd` was also unecessary, so I disabled that
as well. However, eth0 didn't get any IP address at all at that point. I re
enabled `dhcpcd`, and it worked just as expected: one single static IP.

# Setup `netplan`
We configure the `eno1` interface with a static IP. It doesn't need anything
else like DNS since it gets those from wifi. We configure `wlo1' to connect to
an access point and use dhcp. You could override DNS nameservers too if you'd
like. Set `optional: yes` to prevent it blocking on boot up when it can't find
your wifi network.
```
# This is the network config written by 'subiquity'
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      addresses:
        - 192.168.0.1/24
      nameservers:
        search: [lan]
        addresses: [192.168.0.1, 8.8.8.8, 8.8.4.4]
      optional: yes
# if the server is not the router, you have to configure the route to the real
# router with this:
#    routes:
# this didn't work
#      - to: default
#      - to: 0.0.0.0/0
#        via: 192.168.0.2
# This one didn't work for some reason?
#     routes:
#       - to: default
#         via: 192.168.0.2
  wifis:
    wlo1:
      access-points:
        MyWIFI:
          password: "MyPassword"
      dhcp4: true
      optional: yes
#     nameservers:
#       search: [lan]
#       addresses: [192.168.0.1, 8.8.8.8, 8.8.4.4]
```

It's also easy to just copy examples for your specific use case.
```
https://netplan.io/examples
```

Finally, run the follwing.
```
netplan generate
netplan apply
```

The docs suggest that the generate step isn't necessary, but it's warned me
about configuration issues before, so it seems useful to me.

# Setup `systemd-networkd`
If you use `netplan`, don't do this.

Configure it by creating two config files:
```
/etc/systemd/network/10-eth0.network
/etc/systemd/network/11-wlan0.network
```

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

### Match Note
How to match multiple interfaces and use DHCP
```
[Match]
Name=en*

[Network]
DHCP=yes
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

I think `IgnoreCarrierLoss=3s` is a wifi thing. Move it to any interface files
with wifi. It doesn't necessarily go in the external interface configuration.

### DNS Note
We are hosting a DNS server on the internal interface, so we will use it as the
DNS server. That way, we don't have to trust the external network's DNS
settings, and we can use a DNS cache that's on the same machine. It might not be
necessary, but it's probably better.

# `iwd` or `wpa_supplicant`
These'll help you setup the wireless interface. I prefer `iwd` over
wpa_supplicant.

## Wifi strength testing

Generally, you want signal to be above -60 dBm or low packet loss.

### `/proc/net/wireless`

```
cat `/proc/net/wireless`
```
Outputs
```
Inter-| sta-|   Quality        |   Discarded packets               | Missed | WE
 face | tus | link level noise |  nwid  crypt   frag  retry   misc | beacon | 22
  wlo1: 0000   51.  -59.  -256        0      0      0      1    182        0
```
If discarded packets is increasing multiple times a second, you probably have a
bad wifi connection.

### `wavemon`

More info and nicer format, but needs install.
```
wavemon -i wlo1
```

### packet level

Display ip packet statistics
```
ip -s link
```

## Wifi adapter not showing up or stops working after time

### See network interfaces

```
ip link
```
or
```
cd /sys/class/net/
```

### See if the device is recognized as a network adapter
```
lshw -c network
```

### See if the adapter is disabled
```
rfkill list
```

### See if it's connected as a PCI device

Use the following to debug issues:
```
$ lspci -nnk
00:14.3 Network controller [0280]: Intel Corporation Cannon Point-LP CNVi [Wireless-AC] [8086:9df0] (rev 30)
        DeviceName: Onboard - Ethernet
        Subsystem: Intel Corporation Device [8086:4234]
        Kernel driver in use: iwlwifi
        Kernel modules: iwlwifi
```


### See if it's connected as a USB device

```
lsusb
```

### drivers

Search dmesg for issues with the driver or wireless card
```
dmesg | grep -E '(iwlwifi|wlan0)'
```

Google the name from lsusb or lspci followed by linux driver. Hopefully, someone
has something helpful to you.

# `dhcpcd` service

`systemd-networkd` configures the interface to use dhcp, I think `dhcpcd`
actually does the dhcp client work. For that reason, don't disable this service.

# Sources:

A great reference for netplan:
- https://netplan.io/examples

Great resources on systemd-networkd
- https://wiki.archlinux.org/title/systemd-networkd
- https://www.debian.org/doc/manuals/debian-reference/ch05.en.html

A great blog about more advanced config of systemd-networkd
- https://eldon.me/arch-linux-home-router-systemd-configuration/

