---
title: "Disk monitoring with smartctl"
date: 2019-01-16T17:34:37-05:00
draft: false
toc: true
images:
tags: 
  - smart
  - freebsd
  - linux
---

S.M.A.R.T or SMART is a monitoring system built into the majority of modern disk drives. It is very useful to monitor drive health and predict failures.

In order to fully take advantage of SMART on a unix-based system, we will be using an open source tool suite called [smartmontools](https://www.smartmontools.org) which contains the `smartctl` utility. You can install it on FreeBSD using:
```shell
pkg install smartmontools
```
The `smartctl` executable is found under the `sbin` folder, as it is meant for administrative purposes. When using `smartctl`, you should run it under the `root` user.

I will only be dealing with ATA devices for this post (as it's the only type I own), so I'll be excluding any SCSI or NVMe specific info. Additionally, I am using smartmontools 6.6 on FreeBSD.

According to the manpage `smartctl` is a "Control and Monitor Utility for SMART Disks". As always, the manpage is the best place to find a full list of features and descriptions:
```shell
man smartctl
```
However, we will go through few examples and explanations of common commands to get started.

## Scan
The `--scan` flag is useful for finding the drives attached to your system.
```shell
smartctl --scan
```
The output shows I have two disk devices, `/dev/ada0` and `/dev/ada1`.

## Info
The `-i` or `--info` flag provides some useful details about your drive.
```shell
smartctl -i /dev/ada0
```
Here is the output that I get on my system:
```txt
Model Family:     Hitachi/HGST Ultrastar 7K4000
Device Model:     HGST HUS724020ALA640
Serial Number:    PN1134P6HHPTXN
LU WWN Device Id: 5 000cca 22dd53aeb
Firmware Version: MF6OABY0
User Capacity:    2,000,398,934,016 bytes [2.00 TB]
Sector Size:      512 bytes logical/physical
Rotation Rate:    7200 rpm
Form Factor:      3.5 inches
Device is:        In smartctl database [for details use: -P show]
ATA Version is:   ATA8-ACS T13/1699-D revision 4
SATA Version is:  SATA 3.0, 6.0 Gb/s (current: 3.0 Gb/s)
Local Time is:    Fri Jan 18 10:40:53 2019 EST
SMART support is: Available - device has SMART capability.
SMART support is: Enabled
```
Notice that SMART is available and enabled on this drive. If it is not enabled, you can run the `-s on` or `--smart=on` flag to enable it.
```shell
smartctl -s on /dev/ada0
```

## Health
The `-H` or `--health` flag provides a quick status (pass/fail) on your drive:
```shell
smartctl -H /dev/ada0
```
If this command reports a failing health status, then either the device has already failed or its predicted failure is within 24 hours. You should take immediate action if this is the case.

On FreeBSD, you can add the following lines to your `/etc/periodic.conf` to enable a daily health check of your drives:
```txt
daily_status_smart_enable="YES"
daily_status_smart_devices="ada0 ada1"
```

## Attributes
The `-A` or `--attributes` flag provides a set of vendor-defined attributes and their values. These attributes help determine and predict the drives likelihood to fail and its age. It is much more detailed than the health command.
```shell
smartctl -A /dev/ada0
```
Here are my devices attributes:
```txt
ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
  1 Raw_Read_Error_Rate     0x000b   100   100   016    Pre-fail  Always       -       0
  2 Throughput_Performance  0x0005   137   137   054    Pre-fail  Offline      -       78
  3 Spin_Up_Time            0x0007   135   135   024    Pre-fail  Always       -       460 (Average 461)
  4 Start_Stop_Count        0x0012   100   100   000    Old_age   Always       -       108
  5 Reallocated_Sector_Ct   0x0033   100   100   005    Pre-fail  Always       -       0
  7 Seek_Error_Rate         0x000b   100   100   067    Pre-fail  Always       -       0
  8 Seek_Time_Performance   0x0005   142   142   020    Pre-fail  Offline      -       25
  9 Power_On_Hours          0x0012   095   095   000    Old_age   Always       -       40701
 10 Spin_Retry_Count        0x0013   100   100   060    Pre-fail  Always       -       0
 12 Power_Cycle_Count       0x0032   100   100   000    Old_age   Always       -       107
192 Power-Off_Retract_Count 0x0032   100   100   000    Old_age   Always       -       378
193 Load_Cycle_Count        0x0012   100   100   000    Old_age   Always       -       378
194 Temperature_Celsius     0x0002   181   181   000    Old_age   Always       -       33 (Min/Max 18/48)
196 Reallocated_Event_Count 0x0032   100   100   000    Old_age   Always       -       0
197 Current_Pending_Sector  0x0022   100   100   000    Old_age   Always       -       0
198 Offline_Uncorrectable   0x0008   100   100   000    Old_age   Offline      -       0
199 UDMA_CRC_Error_Count    0x000a   200   200   000    Old_age   Always       -       0
```
There is a lot of information to digest here, so we'll go through each tag to form a better understanding (it's not exactly intuitive).
### `ID#` and `ATTRIBUTE_NAME`
<pre>
<mark>ID#</mark> <mark>ATTRIBUTE_NAME</mark>          FLAG     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
194 Temperature_Celsius     0x0002   181   181   000    Old_age   Always       -       33 (Min/Max 18/48)
</pre>
SMART attributes are given an `ID#` from 1 to 253, and are described by and `ATTRIBUTE_NAME`. For example, Attribute 194 provides the temperature (in Celsius) of the disk. Wikipedia provides a decent [list](https://en.wikipedia.org/wiki/S.M.A.R.T.#Known_ATA_S.M.A.R.T._attributes) of common attributes and their descriptions.

### `RAW_VALUE`, `VALUE` and `WORST`
<pre>
ID# ATTRIBUTE_NAME          FLAG     <mark>VALUE</mark> <mark>WORST</mark> THRESH TYPE      UPDATED  WHEN_FAILED <mark>RAW_VALUE</mark>
194 Temperature_Celsius     0x0002   181   181   000    Old_age   Always       -       33 (Min/Max 18/48)
</pre>
The `RAW_VALUE` is the "actual" value of the attribute, for example this disk has a temperature of 33 degrees Celsius. This value is then converted by the disks firmware (not `smartctl`) to a "normalized" `VALUE` from 1-254 (1 is the worst, 254 is the best), which helps determine the state of the disk against a vendor provided threshold. The temperature of the disk is _not_ in fact 181 degrees. The `WORST` value is the lowest or "worst" `VALUE` that has occurred in the past.

### `THRESH` and `WHEN_FAILED`
<pre>
ID# ATTRIBUTE_NAME          FLAG     VALUE WORST <mark>THRESH</mark> TYPE      UPDATED  <mark>WHEN_FAILED</mark> RAW_VALUE
194 Temperature_Celsius     0x0002   181   181   000    Old_age   Always       -       33 (Min/Max 18/48)
</pre>
If the normalized `VALUE` is less than or equal to the threshold value `THRESH`, than the attribute has failed and its `WHEN_FAILED` value will read `FAILING_NOW`. When the `WORST` value is less than or equal to `THRESH`, then the attribute has failed previously and its `WHEN_FAILED` value will read `In_the_past`. Otherwise, it will read `-` (which is the ideal value).

### `TYPE`
<pre>
ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH <mark>TYPE</mark>      UPDATED  WHEN_FAILED RAW_VALUE
194 Temperature_Celsius     0x0002   181   181   000    Old_age   Always       -       33 (Min/Max 18/48)
</pre>
There are two `TYPE` values for attributes:

* A failed `Pre-fail` attribute suggests pending drive failure.
* `Old_age` attributes indicate the aging and wear out of the disk. A failed `Old_age` attribute suggests the project is reaching its end-of-life.

### `UPDATED`
<pre>
ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      <mark>UPDATED</mark>  WHEN_FAILED RAW_VALUE
194 Temperature_Celsius     0x0002   181   181   000    Old_age   Always       -       33 (Min/Max 18/48)
</pre>
There are two possible `UPDATED` values which indicate when the attribute is updated.

* `Always`: Updated during normal operation.
* `Offline`: Updated during an "offline" test.

### `FLAG`
<pre>
ID# ATTRIBUTE_NAME          <mark>FLAG</mark>     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
194 Temperature_Celsius     0x0002   181   181   000    Old_age   Always       -       33 (Min/Max 18/48)
</pre>
Finally we have attribute flags, which are stored under `FLAG`. They describe various properties about each attribute. To get a more readable view of the flags, we can run:
```shell
smartctl -A /dev/ada0 -f brief
```
Which provides this output (`-f brief` typically has a <80 character column width)
```txt
ID# ATTRIBUTE_NAME          FLAGS    VALUE WORST THRESH FAIL RAW_VALUE
  1 Raw_Read_Error_Rate     PO-R--   100   100   016    -    0
  2 Throughput_Performance  P-S---   137   137   054    -    78
  3 Spin_Up_Time            POS---   135   135   024    -    460 (Average 461)
  4 Start_Stop_Count        -O--C-   100   100   000    -    108
  5 Reallocated_Sector_Ct   PO--CK   100   100   005    -    0
  7 Seek_Error_Rate         PO-R--   100   100   067    -    0
  8 Seek_Time_Performance   P-S---   142   142   020    -    25
  9 Power_On_Hours          -O--C-   095   095   000    -    40701
 10 Spin_Retry_Count        PO--C-   100   100   060    -    0
 12 Power_Cycle_Count       -O--CK   100   100   000    -    107
192 Power-Off_Retract_Count -O--CK   100   100   000    -    378
193 Load_Cycle_Count        -O--C-   100   100   000    -    378
194 Temperature_Celsius     -O----   181   181   000    -    30 (Min/Max 18/48)
196 Reallocated_Event_Count -O--CK   100   100   000    -    0
197 Current_Pending_Sector  -O---K   100   100   000    -    0
198 Offline_Uncorrectable   ---R--   100   100   000    -    0
199 UDMA_CRC_Error_Count    -O-R--   200   200   000    -    0
                            ||||||_ K auto-keep
                            |||||__ C event count
                            ||||___ R error rate
                            |||____ S speed/performance
                            ||_____ O updated online
                            |______ P prefailure warning
```
These flags are vendor specific, although according to smartmontool's [sourcecode](https://github.com/smartmontools/smartmontools/blob/20d0ed6f90353ce567003e495a018f7cd5ad149e/smartmontools/atacmds.h#L184-L185) these are "(probably) IBM's, Maxtors and Quantum's definitions for the vendor-specific bits". If the following flag exists for a given attribute, according to [smartlinux](http://smartlinux.sourceforge.net/smart/attributes.php) it most likely means that:

* `K`: The attribute is "self-preserving" and is restored each time when performing a test.
* `C`: Counts events.
* `R`: Measures error rates.
* `S`: Indicates the degradation of speed/performance.
* `O`: The `UPDATED` value is `Always`.
* `P`: The `TYPE` value is `Pre-fail`.

## Capabilities
The `-c` or `--capabilities` flag prints out what SMART features are implemented for this device and how it might react to different SMART commands.
```shell
smartctl -c /dev/ada0
```
Here are the capabilities that my device supports:
```txt
Offline data collection status:  (0x84)	Offline data collection activity
					was suspended by an interrupting command from host.
					Auto Offline Data Collection: Enabled.
Self-test execution status:      (   0)	The previous self-test routine completed
					without error or no self-test has ever 
					been run.
Total time to complete Offline 
data collection: 		(   28) seconds.
Offline data collection
capabilities: 			 (0x5b) SMART execute Offline immediate.
					Auto Offline data collection on/off support.
					Suspend Offline collection upon new
					command.
					Offline surface scan supported.
					Self-test supported.
					No Conveyance Self-test supported.
					Selective Self-test supported.
SMART capabilities:            (0x0003)	Saves SMART data before entering
					power-saving mode.
					Supports SMART auto save timer.
Error logging capability:        (0x01)	Error logging supported.
					General Purpose Logging supported.
Short self-test routine 
recommended polling time: 	 (   1) minutes.
Extended self-test routine
recommended polling time: 	 ( 321) minutes.
SCT capabilities: 	       (0x003d)	SCT Status supported.
					SCT Error Recovery Control supported.
					SCT Feature Control supported.
					SCT Data Table supported.
```
Notice that "Auto Offline Data Collection" is enabled on my drive. This schedules the automatic offline collection of data to be done every four hours, and is useful when the drive has some attributes that are only updated during offline tests. This type of test can technically degrade the performance of the drive, although it is typically only run during periods where the drive is idle. It can be enabled using the `-o VALUE` or `--offlineauto=VALUE` flag.
```shell
smartctl -o on /dev/ada0
```

## Tests
SMART supports running a variety of tests with varying degrees of depth. With the exception of the "offline test", they check the electrical, mechanical and read/write performance of the disk. These can be run using the `--test=` or `-t` flag.

### Offline Test
This runs the "Offline" test immediately which updates [offline attributes](#updated). It is not logged like the other tests.
```shell
smartctl -t offline /dev/ada0
```

### Short Self Test
This test is a short and less thorough version of the self-test. It typically runs in less than ten minutes although it varies by drive. You can find out how long it should take by checking the drives [capabilities](#capabilities) using the `-c` flag. My drive has a short-test time of 1 minutes.
```shell
smartctl -t short /dev/ada0
```

### Extended Self Test
This is the most thorough version of the self-test. It can take tens of minutes to several hours depending on the size and speed of the drive. Again, you can find out how long it should take by checking the drives [capabilities](#capabilities) using the `-c` flag. My drive has a extended-test time of 321 minutes (roughly 5 hours).
```shell
smartctl -t long /dev/ada0
```

### Selective Self Test
This test essentially runs the extended test but **only** on a user defined range of logical block addresses. These are particularly useful for large disks (where extended tests take several hours) or if you suspect a disk is having problems at a particular address. You can run up to five selective tests at once. A full description of the range syntax you can use for the `select` command can be found under the `--test` section of the manpage.
```shell
smartctl -t select,0-1000 -t select,5000-6000 /dev/ada0
```

## Device Logs
Disks with SMART support keep logs containing errors, test results, temperature history and more. A comprehensive and detailed list can be found under the `--log` section of the `smartctl` manpage.

### Errors
You can list the five most recent SMART errors by using the `-l error` or `--log=error` flag:
```shell
smartctl -l error /dev/ada0
```

### Selftest Results
You can list the most recent selftests along with their status and age:
```shell
smartctl -l selftest /dev/ada0
```
Here is what my output looks like:
```txt
Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
# 1  Selective offline   Completed without error       00%     40796         -
# 2  Short offline       Completed without error       00%     40796         -
```
If `LBA_of_first_error` contains a number value, then that is the first block that contains an error. Smartmontools provides a comprehensive [article](https://www.smartmontools.org/wiki/BadBlockHowto) on how to approach these errors.

### Selective Selftest Description
This command displays the start/end logical block addresses for each of the five test spans and their status for the most recent selective selftest.
```shell
smartctl -l selective /dev/ada0
```
This is the log for the selective test I ran [here](#selective-self-test):
```txt
 SPAN  MIN_LBA  MAX_LBA  CURRENT_TEST_STATUS
    1        0     1000  Not_testing
    2     5000     6000  Not_testing
    3        0        0  Not_testing
    4        0        0  Not_testing
    5        0        0  Not_testing
```

## Conclusion
`smartctl` provides an insane amount of functionality and is quite useful for monitoring drive health. Although I haven't outlined all of `smartctl`, this is enough to run a few tests and understand the `-a` or `--all` flag, as `-a` is equivalent to:

* `-H`: [Health](#health)
* `-i`: [Info](#info)
* `-c`: [Capabilities](#capabilities)
* `-A`: [Attributes](#attributes)
* `-l error`: [Error Log](#errors)
* `-l selftest`: [Selftest Log](#selftest-results)
* `-l selective`: [Selective Selftest Log](#selective-selftest-description)

```shell
smartctl -a /dev/ada0
```

As you can probably guess, regularly scheduling tests and scanning for errors can be quite useful. That is where `smartd` comes into play, which I will write about in a future post.
