## About user scripts
While Asuswrt-Merlin only adds a limited number of new features over the original firmware, a lot of customizations can be achieved through the use of user scripts.  These will allow you to setup custom firewall rules, create jobs that can be run at scheduled intervals, or start new services.

Those scripts are stored in the internal non-volatile flash in the [JFFS](https://github.com/RMerl/asuswrt-merlin/wiki/JFFS) partition.  Support for these scripts must be enabled, under Administration -> System on the webui.


## Available scripts:

These shell scripts will be run when certain events occur.  They must be saved in `/jffs/scripts/`.


### services-start
After all other system services have been started at boot.  This is the best place to stop one of these services, and restart it with a different configuration, for example (be aware that any time the service gets manually restarted it will revert back to the original setup however).

### services-stop
Before all system services are stopped, usually on a reboot.

### wan-start
When the WAN interface just came up.  Good place to put scripts that depend on the WAN interface (for example, to update an IPv6 tunnel, or a dynamic DNS).

NOTE: Internet connection is unlikely to be active when this script is run. Add a `sleep` line to delay running until connection complete, or loop until your command succeeds.

### firewall-start
The firewall just got started, and filtering rules have been applied.  This is where you will usually put your own custom rules in the filter table (but NOT the nat table).  Script receives wan interface name (e.g. ppp0) as first parameter.

### nat-start
NAT rules (i.e. port forwards and such) have been applied to the NAT table.  This is where you will want to put your own NAT table custom rules.  For example, a port forward that only allows connections coming from a specific IP.

### init-start
Right after JFFS just got mounted, and before any of the services get start. This is the earliest part of the boot process where you can insert something.

### pre-mount
Just before a partition gets mounted.  This is run in a blocking call and will block the mounting of the  partition for which it is invoked till its execution is complete. This is done so that it can be used for things like running e2fsck on the partition before mounting. This script is also passed the device path (e.g. /dev/sda1) being mounted as an argument which can be used in the script using $1.

### post-mount
Just after a partition got mounted.  $1 is a path where partition has been mounted (e.g. /tmp/mnt/OPT).

### unmount
Just before unmounting a partition.  This is a blocking script, so be careful with it.  The mount point is passed as an argument to the script.

### dhcpc-event
Called whenever a DHCP event occurs on the WAN interface.  The type of event is passed as an argument; possible event types in the version of `udhcpc` in ASUSWRT are `deconfig` (when udhcpc starts and when a lease is lost), `bound` (when a lease and new IP address is acquired), and `renew` (when a lease is renewed, but the IP did not change).

### openvpn-event
Called whenever an OpenVPN server gets started/stopped, or an OpenVPN client connects to a remote server.  Uses the same syntax/parameters as the "up" and "down" scripts in OpenVPN.

### ddns-start
Script called at the end of a DDNS update process.  This script is also called when setting the DDNS type to "Custom".  The script gets passed the WAN IP as an argument.  When handling a "Custom" DDNS, this script is 
also responsible for reporting the success or failure of the update process.  See the [Custom DDNS](https://github.com/RMerl/asuswrt-merlin/wiki/Custom-DDNS) section for more information.

### update-notification
Script called when the scheduled new firmware version availability check detects there's a new firmware available for download. See [update notification example](https://github.com/RMerl/asuswrt-merlin/wiki/update-notification-example) for more info.


## Postconf scripts
Note that in addition to these, you can also use the numerous postconf scripts supported by the firmware, which allows you to execute a script between the moment a service's config file is generated and the service is about to be executed.  See the [Postconf scripts](https://github.com/RMerl/asuswrt-merlin/wiki/Custom-config-files#postconf-scripts) section for more information.


## Creating scripts
Don't forget to set any script you create as being executable:

```
chmod a+rx /jffs/scripts/*
```

And like any Linux script, they need to start with a shebang:

```
#!/bin/sh
```

Also, you must save files with a UNIX encoding.  Note that Windows's Notepad cannot save with a UNIX encoding - get Notepad++ instead.  You can also directly edit them on the router through SSH, by using _vi_ or _nano_, both included in the firmware.  These two will create files already encoded in the proper format.


## Troubleshooting scripts:
Try running your script manually at first to make sure there is no syntax error in it.  You can also insert some code near the top to be able to easily determine if your script did run or not.  For example:

```
touch /tmp/000wanstarted
```
You can then easily tell that the script did run by looking for the presence of 000wanstarted in the /tmp directory.

A useful command for debugging user scripts is `logger`, which will log messages to the system log, visible in the Web UI.
