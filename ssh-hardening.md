# Good guides
- [ssh hardening guide](https://www.ssh-audit.com/hardening_guides.html)
- [another good guide](https://github.com/imthenachoman/How-To-Secure-A-Linux-Server)

# Disable root login over ssh
```
PermitRootLogin no
```

# Disallow `.ssh/authorized_keys2`
```
AuthorizedKeysFile    .ssh/authorized_keys
```

# Better passwords
```
PermitEmptyPasswords no
PubkeyAuthentication yes
```
These are already set by default, so no need to change anything

# Disallow passwords
```
PasswordAuthentication no
```
Require keys to be setup

# Prevent X11 forwarding
```
X11Forwarding no
```

# Allow specific users
```
AllowUsers main_user
```

# Log level
```
LogLevel VERBOSE
```
Then check logs here:
```
journalctl -t sshd -b0
```

# Check login files
```
/etc/issue
/etc/motd
/etc/profile.d
```

# Rhosts
```
IgnoreRhosts yes
HostbasedAuthentication no
```

# Protocol
```
Protocol 2
```

