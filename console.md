# Font Size Too Small

# If you have console-setup installed

In `/etc/default/console-setup`,
```
FONTSIZE="16x32"
```
Then run
```
setupcon
```

# Capslock

## `console-setup`
`/etc/default/keyboard`
```
XKBOPTIONS="ctrl:nocaps"
```
# Arch

Arch uses something [else](https://wiki.archlinux.org/title/Linux_console/Keyboard_configuration): 

Create `/etc/caps2ctrl.kmap`
```
keycode 58 = Control Control Control Control Control Control Control Control Control Control Control Control Control Control Control Control 
```
That's 16 Control's

Then test it with
```
loadkeys /etc/caps2ctrl.kmap
```

Then make it permanent in `/etc/vconsole.conf`
```
KEYMAP=/etc/caps2ctrl.kmap
```

