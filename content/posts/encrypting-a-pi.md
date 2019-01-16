---
title: "Encrypting a Pi"
date: 2017-07-29T16:38:59-05:00
draft: false
toc: true
images:
tags: 
  - linux
  - raspberry-pi
  - arch-linux
  - security
---
### Intro

This week I decided to encrypt my [Arch Linux](https://archlinuxarm.org/platforms/armv7/broadcom/raspberry-pi-2) [Raspberry Pi 2 B](https://www.raspberrypi.org/products/raspberry-pi-2-model-b/).

I used an 64-bit Arch Linux computer for this tutorial, so the steps will likely differ for other Linux distributions.

### Installing Arch Linux Arm
First, plugin a micro SD card. We will follow the [Arch Linux Raspi 2 installation](https://archlinuxarm.org/platforms/armv7/broadcom/raspberry-pi-2) tutorial, but with a few tweaks. Just replace `/dev/sdX` with the device name of the sd card (you can use `sudo fdisk -l` to figure this out).

1. Start `fdisk` to partition the SD card:
```shell
fdisk /dev/sdX
```

2. At the fdisk prompt, delete old partitions and create a new one:
 * Type **o**. This will clear out any partitions on the drive.
 * Type **p** to list partitions. There should be no partitions left.
 * Type **n**, then **p** for primary, **1** for the first partition on the drive, press ENTER to accept the default first sector, then type **+100M** for the last sector.
 * Type **t**, then **c** to set the first partition to type W95 FAT32 (LBA).
 * Type **n**, then **p** for primary, **2** for the second partition on the drive, and  then press ENTER twice to accept the default first and last sector.
Write the partition table and exit by typing **w**.

3. Create and mount the FAT filesystem (You might need to install [dosfstools](https://www.archlinux.org/packages/core/x86_64/dosfstools/)):
```shell
mkfs.vfat /dev/sdX1
mkdir boot
mount /dev/sdX1 boot
```

4. Create and mount the **encrypted** ext4 filesystem:
```shell
cryptsetup luksFormat /dev/sdX2
cryptsetup luksOpen /dev/sdX2 piroot
mkfs.ext4 /dev/mapper/piroot
mkdir root
mount /dev/mapper/piroot root
```

5. Download and extract the root filesystem (as root, not via sudo):
```shell
wget http://os.archlinuxarm.org/os/ArchLinuxARM-rpi-2-latest.tar.gz
bsdtar -xpf ArchLinuxARM-rpi-2-latest.tar.gz -C root
```

### Chroot
Now for the fun part! Since the raspi 2 has `ARMv7` architecture, we need to use [qemu](http://www.qemu.org/) to virtualize it. To do this we need to download the [`qemu`](https://www.archlinux.org/packages/extra/x86_64/qemu/) and [`arch-install-scripts`](https://www.archlinux.org/packages/extra/any/arch-install-scripts/) packages, along with [`binfmt-support`](https://aur.archlinux.org/packages/binfmt-support/) and [`qemu-user-static`](https://aur.archlinux.org/packages/qemu-user-static/) from the [AUR](https://wiki.archlinux.org/index.php/AUR)
```shell
pacaur -S qemu arch-install-scripts binfmt-support qemu-user-static
```
This will allow us to [chroot](https://wiki.archlinux.org/index.php/Change_root) into our raspi's root directory. We will be loosely following [this qemu-chroot tutorial](https://wiki.archlinux.org/index.php/Raspberry_Pi#QEMU_chroot).

1. Start the binfmt-support service
```shell
systemctl daemon-reload
systemctl start binfmt-support
```

2. Make sure that the ARM executable support is active
```shell
update-binfmts --display qemu-arm
```

3. If it is not active, enable it using update-binfmts
```shell
update-binfmts --enable qemu-arm
```

4. Copy the QEMU executables, which will handle the translation from ARM, to the SD card root
```shell
cp /usr/bin/qemu-arm-static root/usr/bin
```

5. Finally chroot into the SD card root
```shell
arch-chroot root/ /bin/bash
```


### Enabling Remote Decryption

Now its time to enable remote decryption, loosely following [this tutorial](https://wiki.archlinux.org/index.php/Dm-crypt/Specialties#Remote_unlocking_of_the_root_.28or_other.29_partition).

1. Update and install basic dependencies:
```shell
pacman -Syu base-devel cower expac git sudo
```
If you get the error `Unsupported ioctl: cmd=0xffffffff80046601` you can ignore it

2. Add user `alarm` to `/etc/sudoers`:
```txt
alarm ALL=(ALL) NOPASSWD: ALL
```
You may want to change this configuration after you complete this tutorial, as it gives the user `alarm` full root permissions

3. Change into user alarm:
```shell
su alarm
```

4. Download and install `pacaur`:
```shell
cd ~
cower -d pacaur
cd pacaur
makepkg --install
```

5. Download and install `ucspi-tcp`:  
This AUR package does not *officially* support `ARMv7`. However it works well on the raspberry pi 2, and other users report it working on other `ARM` based devices. Pass the `-A` flag into `makepkg` to ignore the package architecture. 
```shell
cd ~
cower -d ucspi-tcp
cd ucspi-tcp
makepkg -A --install
```
If you get the error `Unknown host QEMU_IFLA type: 40/41` you can ignore it.

6. Install mkinitcpio hooks:
```shell
pacaur -S mkinitcpio-netconf mkinitcpio-tinyssh mkinitcpio-utils
```

7. Remove build files and exit user `alarm`:
```shell
cd ~
rm -rf .cache/ pacaur/ ucspi-tcp/
exit
```

8. Insert your SSH public key (i.e. the one you usually put onto hosts so that you can ssh in without a password, or a [new one](https://wiki.archlinux.org/index.php/SSH_keys#Generating_an_SSH_key_pair)) into the `/etc/tinyssh/root_key` file.

9. Edit `/etc/mkinitcpio.conf` to add these hooks before `filesystems`:
```txt
HOOKS="... netconf tinyssh encryptssh filesystems ..."
```

10. Regenerate the initramfs image:
```shell
mkinitcpio -p linux-raspberrypi
```

11. Replace `root=/dev/mmcblk0p2` in `/boot/cmdline.txt` with:
```txt
root=/dev/mapper/root cryptdevice=/dev/mmcblk0p2:root ip=:::::eth0:dhcp
```

12. Exit chroot:
```shell
exit
```


### Clean up

Finally, wrap up the disk creation

1. Move boot files to the first partition:
```shell
mv root/boot/* boot
```

2. Remove the `qemu-arm-static` executable:
```shell
rm root/usr/bin/qemu-arm-static
```

3. Sync disks:
```shell
sync
```

4. Unmount disks:
```shell
umount boot root
```

5. Close `/dev/mapper/piroot`:
```shell
cryptsetup luksClose piroot
```


### Connect to pi

Now it's time to connect to the pi

1. Plug in the SD card, ethernet and power.

2. Find out what subnet you are on:
```shell
ip route
```
One of the results should look something like `192.168.#.0/24`

3. Use nmap to find the IP of the pi:
```txt
sudo nmap -sn 192.168.#.0/24
```
One of the responses should look something like this
```txt
Nmap scan report for 192.168.#.#
Host is up (0.0011s latency).
MAC Address: B8:27:EB:FA:F3:3D (Raspberry Pi Foundation)
```

4. Decrypt the pi:
```shell
ssh root@192.168.#.# -i ~/.ssh/my_ssh_key
```
Simply type in the decryption password and you should disconnect

5. Remove the pi's host key from `~/.ssh/known_hosts` (the line containing the pi's IP address). This is because the pi's tinyssh and openssh public key differ, we will fix this soon!

6. Connect to the decrypted pi:
```shell
ssh alarm@192.168.#.#
```
The default password is `alarm`, and the default root password is `root`. You should change these after the tutorial


### Fix Host Keys

1. Install `tinyssh-convert`:
This AUR package does not *officially* support `ARMv7`. However it works well on the raspberry pi 2, and other users report it working on other `ARM` based devices. Pass the `-A` flag into `makepkg` to ignore the package architecture.
```shell
cower -d tinyssh-convert
cd tinyssh-convert
makepkg -A --install
```
        
2. Convert host keys (I'll be using `ed25519` keys):
```shell
sudo tinyssh-convert -f /etc/ssh/ssh_host_ed25519_key -d /etc/tinyssh/sshkeydir/
```

3. Remove `ECDSA` keys:
```shell
sudo rm /etc/tinyssh/sshkeydir/nistp256ecdsa.pk /etc/tinyssh/sshkeydir/.nistp256ecdsa.sk
```

4. Make sure openssh is also using `ed25519` keys:
Edit `/etc/ssh/sshd_config` and uncomment the line
```txt
HostKey /etc/ssh/ssh_host_ed25519_key
```

5. Cleanup downloads:
```shell
cd ~
rm -rf .cache/ tinyssh-convert/
```

6. Regenerate the initramfs image:
```shell
sudo mkinitcpio -p linux-raspberrypi
sudo reboot
```

7. Remove the host key from `~/.ssh/known_hosts` one last time

Remember to use `ssh root@192.168.#.#` for decryption

Now configure your pi normally!


### Bonus: Wireless Decryption

1. Connect to your pi and plugin a wifi adapter

2. Find out what driver your wireless adapter is using:
```shell
readlink /sys/class/net/wlan0/device/driver
```

3. Modify `/etc/mkinitcpio.conf`:
  ```txt
  MODULES="... <driver name>"
  BINARIES="... wpa_passphrase wpa_supplicant"
  HOOKS="... wifi net netconf tinyssh ..."
  ```
  * Add the needed kernel module for your specific wifi adapter.
  * Include the wpa\_passphrase and wpa\_supplicant binaries.
  * Add a hook wifi (or a name of your choice, this is the custom hook that will be created) before the net hook.


4. Create the wifi hook in `/lib/initcpio/hooks/wifi`:
```shell
    run_hook () {
        # sleep a couple of seconds so wlan0 is setup by kernel
        sleep 5
        # set wlan0 to up
        ip link set wlan0 up
        # assocciate with wifi network
        # 1. save temp config file
        wpa_passphrase "network ESSID" "passphrase" > /tmp/wifi
        # 2. assocciate
        wpa_supplicant -B -D nl80211,wext -i wlan0 -c /tmp/wifi
        # sleep a couple of seconds so that wpa_supplicant finishes connecting
        sleep 5
        # wlan0 should now be connected and ready to be assigned an ip by the net hook
    }

    run_cleanuphook () {
        # kill wpa_supplicant running in the background
        killall wpa_supplicant
        # set wlan0 link down
        ip link set wlan0 down
        # wlan0 should now be fully disconnected from the wifi network
    }
```
Make sure to modify "network ESSID" and "passphrase" to match your particular network

5. Create the hook installation file in `/lib/initcpio/install/wifi`:
```shell
build () {
        add_runscript
}
help () {
cat<<HELPEOF
        Enables wifi on boot, for ssh unlocking of disk.
HELPEOF
}
```

6. Replace `ip=:::::eth0:dhcp` in `/boot/cmdline.txt` with:
```txt
ip=:::::wlan0:dhcp
```

7. Regenerate the initramfs image:
```shell
sudo mkinitcpio -p linux-raspberrypi
```


### Sources

* [https://archlinuxarm.org/platforms/armv7/broadcom/raspberry-pi-2](https://archlinuxarm.org/platforms/armv7/broadcom/raspberry-pi-2)
* [https://wiki.archlinux.org/index.php/Raspberry_Pi#QEMU_chroot](https://wiki.archlinux.org/index.php/Raspberry_Pi#QEMU_chroot)
* [https://wiki.archlinux.org/index.php/Dm-crypt/Specialties#Remote_unlocking_of_the_root_.28or_other.29_partition](https://wiki.archlinux.org/index.php/Dm-crypt/Specialties#Remote_unlocking_of_the_root_.28or_other.29_partition)
* [https://wiki.archlinux.org/index.php/Dm-crypt/Specialties#Remote_unlock_via_wifi_.28hooks:_build_your_own.29](https://wiki.archlinux.org/index.php/Dm-crypt/Specialties#Remote_unlock_via_wifi_.28hooks:_build_your_own.29)
* [https://www.offensive-security.com/kali-linux/raspberry-pi-luks-disk-encryption/](https://www.offensive-security.com/kali-linux/raspberry-pi-luks-disk-encryption/)
