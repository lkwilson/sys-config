# Bridging in the context of setting up a [router](router.md):
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

# How to make a bridge
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
