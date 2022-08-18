# Basic hardening
[ssh hardening guide](https://www.ssh-audit.com/hardening_guides.html)

# Disable root login over ssh
```
< #PermitRootLogin prohibit-password
---
> PermitRootLogin no
```

# Disallow `.ssh/authorized_keys2`
```
< #AuthorizedKeysFile   .ssh/authorized_keys .ssh/authorized_keys2
---
> AuthorizedKeysFile    .ssh/authorized_keys
```

# Better passwords
```
< #PermitEmptyPasswords no
< #PubkeyAuthentication yes
---
> PermitEmptyPasswords no
> PubkeyAuthentication yes
```
These are already set by default, so no need to change anything

# Disallow passwords
```
PasswordAuthentication no
```
Require keys

# Allow X11 forwarding
```
< #X11Forwarding no
---
> X11Forwarding yes
```


