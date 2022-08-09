# `ip` cheat sheet

```
ip addr show [ dev STRING ]
ip addr { add | del } PREFIX dev STRING

ip link set DEVICE { up | down }
ip link show [ DEVICE ]
ip link set [ dev ] DEVICE { up | down }

ip neighbour

ip route
ip route { add | del } { default | PREFIX } { via ADDRESS | dev STRING }
```

- you don't have to type the whole name
- deletes work on the first, longest match
- `PREFIX`: `192.168.1.1/24`
- `STRING|DEVICE`: `eth0`
- `ADDRESS`: `192.168.1.1`
