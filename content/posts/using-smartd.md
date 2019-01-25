---
title: "Using smartd"
date: 2019-01-21T11:10:07-05:00
draft: false
toc: true
images:
tags: 
  - smart
  - freebsd
  - linux
---

_**Note**: This article is a continuation of [Disk monitoring with smartctl](/posts/disk-monitoring-with-smartctl/)_

As mentioned in my previous post, S.M.A.R.T or SMART is a monitoring system built into the majority of modern disk drives. It is very useful to monitor drive health and predict failures.

The open-source tool suite [smartmontools](https://www.smartmontools.org) contains the monitoring daemon `smartd` which can be used to collect and report disk information. Although I will be covering much of the configuration and functionality of `smartd`, the manpages are always useful:
```shell
man smartd
man smartd.conf
```

Again, I will only be dealing with ATA devices for this post (as it’s the only type I own), so I’ll be excluding any SCSI or NVMe specific info. Additionally, I am using smartmontools 6.6 on FreeBSD.

## Setup
The following setup steps are specific to FreeBSD, although should be similar enough on other unix-based systems.

First, copy the sample configuration file:
```shell
cp /usr/local/etc/smartd.conf.sample /usr/local/etc/smartd.conf
```
By default `smartd` uses the `syslog` facility `daemon`, which is not handled by default on FreeBSD. For this example, we will log this facility to `/var/log/daemon.log`  
Create `/usr/local/etc/syslog.d/daemon.conf` with the following contents:
```txt
daemon.*                                        /var/log/daemon.log
```
Now create the logfile and restart `syslogd`
```shell
touch /var/log/daemon.log
chmod 600 /var/log/daemon.log
service syslogd restart
```
We can now enable and start `smartd`:
```shell
service smartd enable
service smartd start
```
If everything has gone well, `/var/log/daemon.log` should contain something similar to this:
```txt
Jan 21 12:13:21 coruscant smartd[87016]: smartd 6.6 2017-11-05 r4594 [FreeBSD 12.0-RELEASE-p2 amd64] (local build)
Jan 21 12:13:21 coruscant smartd[87016]: Copyright (C) 2002-17, Bruce Allen, Christian Franke, www.smartmontools.org
Jan 21 12:13:21 coruscant smartd[87016]: Opened configuration file /usr/local/etc/smartd.conf
Jan 21 12:13:21 coruscant smartd[87016]: Drive: DEVICESCAN, implied '-a' Directive on line 23 of file /usr/local/etc/smartd.conf
Jan 21 12:13:21 coruscant smartd[87016]: Configuration file /usr/local/etc/smartd.conf was parsed, found DEVICESCAN, scanning devices
Jan 21 12:13:21 coruscant smartd[87016]: Device: /dev/ada0, opened
Jan 21 12:13:21 coruscant smartd[87016]: Device: /dev/ada0, HGST HUS724020ALA640, S/N:PN1134P6HHPTXN, WWN:5-000cca-22dd53aeb, FW:MF6OABY0, 2.00 TB
Jan 21 12:13:21 coruscant smartd[87016]: Device: /dev/ada0, found in smartd database: Hitachi/HGST Ultrastar 7K4000
Jan 21 12:13:23 coruscant smartd[87016]: Device: /dev/ada0, is SMART capable. Adding to "monitor" list.
Jan 21 12:13:23 coruscant smartd[87016]: Device: /dev/ada1, opened
Jan 21 12:13:23 coruscant smartd[87016]: Device: /dev/ada1, HGST HUS724020ALA640, S/N:PN1134P6HJ3RTN, WWN:5-000cca-22dd56b76, FW:MF6OABY0, 2.00 TB
Jan 21 12:13:23 coruscant smartd[87016]: Device: /dev/ada1, found in smartd database: Hitachi/HGST Ultrastar 7K4000
Jan 21 12:13:24 coruscant smartd[87016]: Device: /dev/ada1, is SMART capable. Adding to "monitor" list.
Jan 21 12:13:24 coruscant smartd[87016]: Monitoring 2 ATA/SATA, 0 SCSI/SAS and 0 NVMe devices
Jan 21 12:13:27 coruscant smartd[87018]: smartd has fork()ed into background mode. New PID=87018.
Jan 21 12:13:27 coruscant smartd[87018]: file /var/run/smartd.pid written containing PID 87018
```

## Devices
`smartd` can be given a list of devices to monitor. Each device can be given a list of directives (flags) that determine how `smartd` interacts with it. If no directives are added, `smartd` defaults to the `-a` directive. Additionally, there are a couple s (`DEVICESCAN` and `DEFAULT`) that can be used alongside/instead of the devices list.

For example, I can write this to my `smartd.conf`:
```txt
/dev/ada0
/dev/ada1 -a -m root
```
In this case, `/dev/ada0` defaults to having the `-a` directive, and `/dev/ada1` is given the `-a` and `-m root` directives.

### `DEVICESCAN`
`DEVICESCAN` represents all devices and applies its directives to each device.
For example:
```txt
DEVICESCAN -a -m root
```
Is equivalent to (on my system)
```txt
/dev/ada0 -a -m root
/dev/ada1 -a -m root
```

### `DEFAULT`
All directives passed into `DEFAULT` are applied to each device below it. Here is an example from the manpage:  
This configuration:
```txt
DEFAULT -a -R5! -W 2,40,45 -I 194 -s L/../../7/00 -m admin@example.com
/dev/sda
/dev/sdb
/dev/sdc
DEFAULT -H -m admin@example.com
/dev/sdd
/dev/sde -d removable
```
Has the same effect as:
```txt
/dev/sda -a -R5! -W 2,40,45 -I 194 -s L/../../7/00 -m admin@example.com
/dev/sdb -a -R5! -W 2,40,45 -I 194 -s L/../../7/00 -m admin@example.com
/dev/sdc -a -R5! -W 2,40,45 -I 194 -s L/../../7/00 -m admin@example.com
/dev/sdd -H -m admin@example.com
/dev/sde -d removable -H -m admin@example.com
```

## Directives
Running `smartd -D` provides a simplified list of directives:
<pre>
-d TYPE Set the device type: auto, ignore, removable,
        ata, scsi, nvme[,NSID], sat[,auto][,N][+TYPE], usbcypress[,X], usbjmicron[,p][,x][,N], usbprolific, usbsunplus, intelliprop,N[+TYPE], 3ware,N, hpt,L/M/N, cciss,N, areca,N/E, atacam
-T TYPE Set the tolerance to one of: normal, permissive
<a href="#data-collection">-o VAL  Enable/disable automatic offline tests (on/off)
-S VAL  Enable/disable attribute autosave (on/off)</a>
-n MODE No check if: never, sleep[,N][,q], standby[,N][,q], idle[,N][,q]
-H      Monitor SMART Health Status, report if failed
<a href="#test-scheduling">-s REG  Do Self-Test at time(s) given by regular expression REG</a>
<a href="#log-errors">-l TYPE Monitor SMART log or self-test status:
        error, selftest, xerror, offlinests[,ns], selfteststs[,ns]</a>
-l scterc,R,W  Set SCT Error Recovery Control
-e      Change device setting: aam,[N|off], apm,[N|off], dsn,[on|off],
        lookahead,[on|off], security-freeze, standby,[N|off], wcache,[on|off]
-f      Monitor 'Usage' Attributes, report failures
<a href="#mail-alerts">-m ADD  Send email warning to address ADD</a>
-M TYPE Modify email warning behavior (see man page)
-p      Report changes in 'Prefailure' Attributes
-u      Report changes in 'Usage' Attributes
-t      Equivalent to -p and -u Directives
<a href="#raw-values">-r ID   Also report Raw values of Attribute ID with -p, -u or -t</a>
-R ID   Track changes in Attribute ID Raw value with -p, -u or -t
-i ID   Ignore Attribute ID for -f Directive
<a href="#ignore">-I ID   Ignore Attribute ID for -p, -u or -t Directive</a>
-C ID[+] Monitor [increases of] <a href="#current-pending-sectors">Current Pending Sectors</a> in Attribute ID
-U ID[+] Monitor [increases of] <a href="#offline-uncorrectable-sectors">Offline Uncorrectable Sectors</a> in Attribute ID
<a href="#temperature">-W D,I,C Monitor Temperature D)ifference, I)nformal limit, C)ritical limit</a>
-v N,ST Modifies labeling of Attribute N (see man page)
-P TYPE Drive-specific presets: use, ignore, show, showall
-a      Default: -H -f -t -l error -l selftest -l selfteststs -C 197 -U 198
-F TYPE Use firmware bug workaround:
        none, nologdir, samsung, samsung2, samsung3, xerrorlba
 #      Comment: text after a hash sign is ignored
 \      Line continuation character
</pre>

### Data collection
`-S on` enables attribute autosave (necessary for ['always updated' tests](/posts/disk-monitoring-with-smartctl/#updated)) and `-o on` enables automatic offline tests (necessary for ['offline updated' tests](/posts/disk-monitoring-with-smartctl/#updated)).
```shell
/dev/ada0 -o on -S on
```
You can also use `smartctl` to enable these on a drive and it should persist between reboots.
```shell
smartctl -o on -S on /dev/ada0`
```

### Log errors
`-l` reports increases in the number of errors in SMART logs.

* `-l error` reports when the number of ATA errors increases since the last check.
* `-l selftest` reports when the number of selftest log errors increases since the last check.
* `-l selfteststs` report changes to the selftest executions status.

### Mail alerts
You can subscribe to `smartd` email alerts by using the `-m email1,email2` directive. This can be configured further using the `-M` directive (look it up in the manpage). This sends and email if the following directives detect a failure or a new error:

* `-H`: Health
* `-l`: [Log Errors](#log-errors)
* `-f`: Usage Attribute Failure
* `-C`: Report [Current Pending Sectors](#current-pending-sectors)
* `-U`: Report [Offline Uncorrectable Sectors](#offline-uncorrectable-sectors)

This example sends alerts to the `root` user:
```txt
/dev/ada0 -a -m root
```

### Raw values
`-r` reports each change to an attribute in it's "raw" value form. For example, if `-r 194` is used and the disk temperature changes, `smartd` will output the actual change in Celsius rather than the change in the vendor-defined value. This can be useful for more "tangible" attributes. Here is what `-r 194` might look like:
```txt
Jan 22 22:43:29 coruscant smartd[87018]: Device: /dev/ada1, SMART Usage Attribute: 194 Temperature_Celsius changed from 181 [Raw 33 (Min/Max 18/48)] to 176 [Raw 34 (Min/Max 18/48)]
Jan 23 00:43:28 coruscant smartd[87018]: Device: /dev/ada0, SMART Usage Attribute: 194 Temperature_Celsius changed from 181 [Raw 33 (Min/Max 18/48)] to 176 [Raw 34 (Min/Max 18/48)]
Jan 23 02:43:29 coruscant smartd[87018]: Device: /dev/ada1, SMART Usage Attribute: 194 Temperature_Celsius changed from 176 [Raw 34 (Min/Max 18/48)] to 171 [Raw 35 (Min/Max 18/48)]
Jan 23 03:13:28 coruscant smartd[87018]: Device: /dev/ada0, SMART Usage Attribute: 194 Temperature_Celsius changed from 176 [Raw 34 (Min/Max 18/48)] to 171 [Raw 35 (Min/Max 18/48)]
Jan 23 03:43:28 coruscant smartd[87018]: Device: /dev/ada0, SMART Usage Attribute: 194 Temperature_Celsius changed from 171 [Raw 35 (Min/Max 18/48)] to 176 [Raw 34 (Min/Max 18/48)]
Jan 23 04:43:28 coruscant smartd[87018]: Device: /dev/ada0, SMART Usage Attribute: 194 Temperature_Celsius changed from 176 [Raw 34 (Min/Max 18/48)] to 171 [Raw 35 (Min/Max 18/48)]
Jan 23 05:43:27 coruscant smartd[87018]: Device: /dev/ada0, SMART Usage Attribute: 194 Temperature_Celsius changed from 171 [Raw 35 (Min/Max 18/48)] to 176 [Raw 34 (Min/Max 18/48)]
Jan 23 06:13:27 coruscant smartd[87018]: Device: /dev/ada0, SMART Usage Attribute: 194 Temperature_Celsius changed from 176 [Raw 34 (Min/Max 18/48)] to 171 [Raw 35 (Min/Max 18/48)]
Jan 23 09:43:28 coruscant smartd[87018]: Device: /dev/ada1, SMART Usage Attribute: 194 Temperature_Celsius changed from 171 [Raw 35 (Min/Max 18/48)] to 166 [Raw 36 (Min/Max 18/48)]
```
Here is how you would enable this:
```
/dev/ada0 -r 194
```

### Ignore
`-I` ignores a particular attribute and modifies the behavior of `-p`, `-u` and `-t`. This is useful for certain attributes which fluctuate often, such as disk temperature. For example, here is a snippet of my `smartd` log:
```txt
Jan 21 15:13:28 coruscant smartd[87018]: Device: /dev/ada1, SMART Usage Attribute: 194 Temperature_Celsius changed from 187 to 181
Jan 21 17:13:28 coruscant smartd[87018]: Device: /dev/ada0, SMART Usage Attribute: 194 Temperature_Celsius changed from 187 to 181
Jan 21 18:13:28 coruscant smartd[87018]: Device: /dev/ada0, SMART Usage Attribute: 194 Temperature_Celsius changed from 181 to 187
Jan 21 18:43:28 coruscant smartd[87018]: Device: /dev/ada0, SMART Usage Attribute: 194 Temperature_Celsius changed from 187 to 181
Jan 21 19:13:28 coruscant smartd[87018]: Device: /dev/ada0, SMART Usage Attribute: 194 Temperature_Celsius changed from 181 to 187
Jan 21 20:43:29 coruscant smartd[87018]: Device: /dev/ada1, SMART Usage Attribute: 194 Temperature_Celsius changed from 181 to 187
Jan 22 01:43:27 coruscant smartd[87018]: Device: /dev/ada0, SMART Usage Attribute: 194 Temperature_Celsius changed from 187 to 193
Jan 22 02:13:28 coruscant smartd[87018]: Device: /dev/ada1, SMART Usage Attribute: 194 Temperature_Celsius changed from 187 to 193
Jan 22 07:13:27 coruscant smartd[87018]: Device: /dev/ada0, SMART Usage Attribute: 194 Temperature_Celsius changed from 193 to 200
Jan 22 07:43:28 coruscant smartd[87018]: Device: /dev/ada1, SMART Usage Attribute: 194 Temperature_Celsius changed from 193 to 200
Jan 22 09:43:28 coruscant smartd[87018]: Device: /dev/ada0, SMART Usage Attribute: 194 Temperature_Celsius changed from 200 to 193
Jan 22 09:43:29 coruscant smartd[87018]: Device: /dev/ada1, SMART Usage Attribute: 194 Temperature_Celsius changed from 200 to 193
```
As you can see, it's useful to add the `-I` directive in order to ignore temperature changes (attribute 194 or 231):
```
/dev/ada0 -I 194
```

### Current pending sectors
A pending sector is "unstable" and is waiting to be remapped due to an unrecoverable read error. It is not remapped immediately, but on the next write to that sector.

### Offline uncorrectable sectors
A disk sector which is found to be unreadable during an offline test or selftest.

### Temperature
The `-W DIFF[,INFO[,CRIT]]` directive provides some extra functionality for monitoring temperature attributes. The manpage examples do a good job of describing it's functionality:

To track temperature changes of at least 2 degrees, use:
```txt
-W 2
```
To log informal messages on temperatures of at least 40 degrees,
use:
```txt
-W 0,40
```
For warning messages/mails on temperatures of at least 45
degrees, use:
```txt
-W 0,0,45
```
To combine all of the above reports, use:
```txt
-W 2,40,45
```

## Test scheduling
There are many [tests](/posts/disk-monitoring-with-smartctl/#tests) that can be run against SMART devices, scheduling them using `smartd` is quite easy! You must add the following directive to a drive:
```txt
/dev/ada0 -s T/MM/DD/d/HH
```

The `T` parameter is a single letter which denotes the type of test to be run:

* `L`: [Extended Self Test](/posts/disk-monitoring-with-smartctl/#extended-self-test)
* `S`: [Short Self Test](/posts/disk-monitoring-with-smartctl/#short-self-test)
* `O`: [Immediate Offline Test](/posts/disk-monitoring-with-smartctl/#offline-test)
* `n`: Run the next [Selective Self Test](/posts/disk-monitoring-with-smartctl/#selective-self-test)
* `r`: Redo the most recent [Selective Self Test](/posts/disk-monitoring-with-smartctl/#selective-self-test)
* `c`: If the most recent [Selective Self Test](/posts/disk-monitoring-with-smartctl/#selective-self-test) failed, redo it. Otherwise, run the next one.

The `MM/DD/d/HH` matches for date/time:

* `MM`: **Two digit** month from 01-12 (Jan-Dec) inclusive.
* `DD`: **Two digit** date from 01-31 inclusive.
* `d`: One digit day from 1-7 (Mon-Sun) inclusive.
* `HH`: **Two digit** hour from 00-23 inclusive.

When specifying a parameter, you can denote _every_ value with a dot (`.`), multiple values in brackets (`(A|B|C)` represents values A and B and C), and a range of values in square brackets (`[A-B]` represents all values from A to B). If you need more advanced matching, look at the `-s` section of the manpage.

From the manpage we have the following examples which better describe how to schedule tests:

To schedule a short Self-Test between 2-3 am every morning, use:
```txt
-s S/../.././02
```
To schedule a long Self-Test between 4-5 am every Sunday morning, use:
```txt
-s L/../../7/04
```
To schedule a long Self-Test between 10-11 pm on the first and fifteenth day of each month, use:
```txt
-s L/../(01|15)/./22
```
To schedule an Offline Immediate test after every midnight, 6 am, noon, and 6 pm, plus a Short Self-Test daily at 1-2 am and a Long Self-Test every Saturday at 3-4 am, use:
```txt
-s (O/../.././(00|06|12|18)|S/../.././01|L/../../6/03)
```
If Long Self-Tests of a large disks take longer than the system uptime, a full disk test can be performed by several Selective Self-Tests.  To setup a full test of a 1 TB disk within 20 days
(one 50 GB span each day), run this command once:
```shell
smartctl -t select,0-99999999 /dev/sda
```
To run the next test spans on Monday-Friday between 12-13 am, run `smartd` with this directive:
```txt
-s n/../../[1-5]/12
```

## `smartd` flags
There a couple useful flags for the `smartd` command. Again, consult the manpage for more details. On FreeBSD, simply add these flags to the `smartd_flags` variable in your `rc.conf`.

### Interval
The interval at which `smartd` scans the disk attributes can be configured using the `-i N` flag where `N` is in seconds. The default is 1800 (or thirty minutes).
```txt
-i 1800
```

### Store disk attributes
You can store disk attributes to a file on every interval to a csv file using the `-A path` flag:
```txt
-A /var/log/smartd/
```
The csv format is documented on the `smartd` manpage.

## Conclusion
`smartd` is a useful daemon for monitoring SMART attributes and reporting errors/issues. My `smartd.conf` currently looks like this:
```txt
DEFAULT -a -o on -S on -m root -W 2,40,45 -R 194 -s S/../.././02
/dev/ada0
/dev/ada1
```
