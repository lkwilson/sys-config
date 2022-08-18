I installed openwrt on an rpi 4

# VLAN
I plugged the PI into a managed switch, and set the port to have a native vlan
for the lan and a tagged vlan for the wan. I added an interface based off eth0,
and used 802.1q. 802.1ad might work, but I got it working with 802.1q first.

# POE HAT

```
opkg update
opkg install kmod-hwmon-rpi-poe-fan
```
I also installed `kmod-pwm-bcm2835`, but not sure if it was necessary.

Then,
```/boot/config.txt
...
dtoverlay=rpi-poe
dtparam=poe_fan_temp0=50000
#dtparam=poe_fan_temp0_hyst=2000
dtparam=poe_fan_temp1=55000
#dtparam=poe_fan_temp1_hyst=2000
dtparam=poe_fan_temp2=60000
#dtparam=poe_fan_temp2_hyst=2000
dtparam=poe_fan_temp3=65000
#dtparam=poe_fan_temp3_hyst=5000
```

# DMZ

You add another tagged vlan to the port, and this can become your dmz. Create a
network interface in the same way as above. Edit the firewall for the following:
## input: reject, but allow dhcp, dns, icmp, and igmp

This prevents things on the dmz from talking to the router except the excepted
protocols.

## output: accept

This allows the rpi to talk to devices on the dmz

## forward: reject, allow forward to wan, allow forward from lan

This allows the dmz to talk to the internet, and allows things from the lan to
talk to things on the dmz. DMZ cannot forward to lan.
