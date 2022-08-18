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
## If you have whatever arch installs

It's wiki will tell you how to configure it, but if your control stays on after
you release capslock, you have to add 16 space separated new mapping values.

I forget what it was exactly, but I have it noted somewhere.

TODO
