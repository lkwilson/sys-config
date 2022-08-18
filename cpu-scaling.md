You can scale the cpu frequency down to preserve power:

cpupower command

# System Info
Get information about cpus with this:
```
$ sudo cpupower -c all frequency-info
```
Important information to note:
```
...
  hardware limits: 400 MHz - 4.60 GHz
  available cpufreq governors: performance powersave
  current CPU frequency: 4.13 GHz (asserted by call to kernel)
...
```
We then will use the following environment variables for the commands below:
```
HIGH=4.60GHz
LOW=400MHz
```

# max performance, no power saving, any cpu frequency
```
sudo cpupower frequency-set -g performance
sudo cpupower frequency-set -u $HIGH
sudo cpupower frequency-set -d $LOW
```

# normal performance, uses power saving, any cpu frequency
```
sudo cpupower frequency-set -g powersave
sudo cpupower frequency-set -u $HIGH
sudo cpupower frequency-set -d $LOW
```

# low performance, uses power saving, only low cpu frequency
```
sudo cpupower frequency-set -g powersave
sudo cpupower frequency-set -u $LOW
sudo cpupower frequency-set -d $LOW
```
