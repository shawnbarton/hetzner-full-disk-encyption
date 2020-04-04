This is a translation and adaptation of the instructions found at https://seeseekey.net/archive/122144/. 

To Do:
1. Translate more explanation text
2. Modify with Cloudflare dm-crypt

Pre-requisites:
1. A registered SSH key in Hetzner's "Robot" Web Interface Key Management

Guide:
1\) Boot the server into Rescue Mode via the Hetzner "Robot" Web Interface.

2\) Optional - Put your pre-configured installimage config into ´/autosetup´ using nano or vi

Sample config:
```bash
##  Hetzner Online GmbH - installimage - config

##  HARD DISK DRIVE(S):
# Onboard: ST4000NM0024-1HT178
DRIVE1 /dev/sda
# Onboard: ST4000NM0024-1HT178
DRIVE2 /dev/sdb

##  SOFTWARE RAID:
## activate software RAID?  < 0 | 1 >
SWRAID 1

## Choose the level for the software RAID < 0 | 1 | 10 >
SWRAIDLEVEL 0

##  BOOTLOADER:
BOOTLOADER grub

##  HOSTNAME:
HOSTNAME server

##  PARTITIONS / FILESYSTEMS:
PART /boot  ext3     2G
PART lvm    vg0       all

LV vg0   swap   swap     swap         64G
LV vg0   root   /        ext4         all

##  OPERATING SYSTEM IMAGE:
IMAGE /root/.oldroot/nfs/install/../images/Ubuntu-1804-bionic-64-minimal.tar.gz
```

2.1\) Run the command `installimage`. If a config is not already in place then it will prompt you to create one with some on-screen menus.

3\) Once the installation completes, reboot the server so that it boots into the newly installed operating system with the command `reboot`.

4\) While the server is rebooting, create a new ssh key pair on your **local machine**. For example, using the command `ssh-keygen -t rsa -b 4096 -f .ssh/dropbear`.

5\) Reconnect to the server (using your normal SSH key).

5.1\) You may receive the error `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`. To fix this, you will need to remove the server fingerprint from your known_hosts file.

6\) Once connected, update your apt repositories and upgrade any packages on the server using the command:
```bash
apt-get update && apt-get dist-upgrade
```

6.1\) Install busybox and dropbear using the command:
```bash
apt-get install busybox dropbear-initramfs
```

7\) Edit the initramfs configuration using the following command:
```bash
nano /etc/initramfs-tools/initramfs.conf
```

7.1\) Replace the line:
```bash
BUSYBOX=auto
```
with:
```bash
BUSYBOX=y
```

8\) Copy your SSH public key (generated in step 4, probably as .ssh/dropbear.pub, on your local system) into the dropbear authorized_keys file:
```bash
nano /etc/dropbear-initramfs/authorized_keys
```

You can have multiple keys in this file. Using a separate SSH key for dropbear is optional. You can also use the same SSH key that you use to log in to the server.

9\) **Ubuntu 18.04 ONLY** In Ubuntu 18.04, there is a bug preventing drive decryption which requires us to apply the following workaround:
```bash
cd /etc/initramfs-tools/hooks/
nano cryptsetup-fix.sh
```

Paste the following script into cryptsetup-fix.sh:
```bash
#!/bin/sh

# This hook is for fixing busybox-initramfs issue while unlocking a luks
# encrypted rootfs. The problem is that the included busybox version
# is stripped down to the point that it breaks cryptroot-unlock script:
# https://bugs.launchpad.net/ubuntu/+source/busybox/+bug/1651818

# This is a non-aggressive fix based on the original busybox-initramfs hook
# until the bug is fixed.
# busybox or busybox-static package must be present for this to work

# This file should be placed in /etc/initramfs-tools/hooks/ and have +x flag set
# after that you need to rebuild the initramfs with 'update-initramfs -u'

# Users reported the solution working on at least:
# Ubuntu 17.04, 17.10, 18.04

# Also note that this does not replace busybox-initramfs package.
# The package must be present, this hook just fixes what's broken.

# Hamy - www.hamy.io

set -e

case "${1:-}" in
  prereqs)  echo ""; exit 0;;
esac

[ n = "$BUSYBOX" ] && exit 0

[ -r /usr/share/initramfs-tools/hook-functions ] || exit 0
. /usr/share/initramfs-tools/hook-functions

# Testing the presence of busybox-initramfs hook
[ -x /usr/share/initramfs-tools/hooks/zz-busybox-initramfs ] || exit 0

# The original busybox binary added by busybox-initramfs
BB_BIN_ORG=$DESTDIR/bin/busybox
[ -x $BB_BIN_ORG ] || exit 0

# The one we want to replace it with
[ -x /bin/busybox ] || exit 0
BB_BIN=/bin/busybox

# Ensure the original busybox lacks extended options
# and the soon-to-be-replaced-by one does not
if $BB_BIN_ORG ps -eo pid,args >/dev/null 2>&1; then
  exit 0
elif ! $BB_BIN ps -eo pid,args >/dev/null 2>&1; then
  exit 0
fi

# Get the inode number of busybox-initramfs binary
BB_BIN_ORG_IND=$(stat --format=%i $BB_BIN_ORG)

# Replace the binary
rm -f $BB_BIN_ORG
copy_exec $BB_BIN /bin/busybox

echo -n "Fixing busybox-initramfs for:"

for alias in $($BB_BIN --list-long); do
  alias="${alias#/}"
  case "$alias" in
    # strip leading /usr, we don't use it
    usr/*) alias="${alias#usr/}" ;;
    */*) ;;
    *) alias="bin/$alias" ;;  # make it into /bin
  esac

  # Remove (and then re-add) all the hardlinks added by busybox-initramfs
  if [ -e "$DESTDIR/$alias" ] && [ $(stat --format=%i "$DESTDIR/$alias") -eq $BB_BIN_ORG_IND ]; then
      echo -n " ${alias##*/}"
      rm -f "$DESTDIR/$alias"
      ln "$DESTDIR/bin/busybox" "$DESTDIR/$alias"
  fi
done

# To get a trailing new line
echo
```

... and make the script executable:

```bash
chmod +x cryptsetup-fix.sh
```

10\) Once more, enable **rescue mode** in Hetzner's "Robot" Web Interface and reboot the server:
```bash
reboot
```

Reconnect to the server using SSH once it has booted into rescue mode.

11\) We must now mount the LVM, since these are not automatically mounted to `/dev/mapper/`, we run the following commands:
```bash
lvm vgscan -v
lvm vgchange -a y
```

This will scan for existing Volume Groups and activate them.

12\) We can now mount the LVM using the command:
```bash
mount /dev/mapper/vg0-root /mnt/
```

13\) Copy the existing Ubuntu installation, this may take a little while:
```bash
mkdir /oldroot/
rsync -a /mnt/ /oldroot/
```

14\) Unmount LVM:
```bash
umount /mnt/
```

15\) We will now delete the existing Volume Group:
```bash
vgremove vg0
```
You will see a series of prompts, to which you must answer YES, for example:
```bash
Do you really want to remove volume group "vg0" containing 2 logical volumes? [y/n]: y
Do you really want to remove active logical volume swap? [y/n]: y
  Logical volume "swap" successfully removed
Do you really want to remove active logical volume root? [y/n]: y
  Logical volume "root" successfully removed
  Volume group "vg0" successfully removed
```
16\) Now that we have deleted the Volume Group, we will create a new and encrypted dm-crypt device:
```bash
cryptsetup --cipher aes-xts-plain64 --key-size 256 --hash sha256 --iter-time=10000 luksFormat /dev/md1
```

During this process, you will be prompted to confirm that any existing data is deleted permanently and to enter your drive encryption passphrase (key):

```bash
WARNING!
========
This will overwrite data on /dev/md1 irrevocably.

Are you sure? (Type uppercase yes): YES
Enter passphrase:
Verify passphrase:
```

17\) "Open" the newly created dm-crypt device:
```bash
cryptsetup luksOpen /dev/md1 cryptroot
```

You will be asked for your encryption key.

18\) Create a Physical Volume for "cryptroot":
```bash
pvcreate /dev/mapper/cryptroot
```

19\) Create a new Volume Group:
```bash
vgcreate vg0 /dev/mapper/cryptroot

lvcreate -n swap -L64G vg0
lvcreate -n root -l100%FREE vg0
```

Be sure to adapt this to any customizations you expect to the original table provided with installimage in Step 2.

20\) Create the Filesystem on each Logical Volume (corresponding to the Volume Groups in Step 19):
```bash
mkfs.ext4 /dev/vg0/root
mkswap /dev/vg0/swap
```

21\) We will now restore our operating system onto the empty encrypted volumes:
```bash
mount /dev/vg0/root /mnt/
rsync -a /oldroot/ /mnt/
```

22\) We will now prepare the system for a chroot procedure. This allows us to assume the context of the installed operating system and run commands, even while we are still inside the rescue system.

```bash
mount /dev/md0 /mnt/boot
mount --bind /dev /mnt/dev
mount --bind /sys /mnt/sys
mount --bind /proc /mnt/proc
chroot /mnt
```

23\) We are now within the context of the installed operating system. We must configure it to recognize our encrypted block device:
```bash
nano /etc/crypttab
```

Add the following line:
```bash
cryptroot /dev/md1 none luks
```

24\) We will now update initramfs with the following commmand:
```bash
update-initramfs -u
```

You may receive the following error:
```bash
dropbear: WARNING: Invalid authorized_keys file, remote unlocking of cryptroot via SSH won't work!
```

This means that your authorized_keys file is in the wrong location. Fix this before proceeding.

25\) Once initramfs has been successfully updated, we will update the grub bootloader:

```bash
update-grub
grub-install /dev/sda
grub-install /dev/sdb
```

During `update-grub` you may receive the following warnings and errors:

```bash
Generating grub configuration file ...
  WARNING: Failed to connect to lvmetad. Falling back to device scanning.
  WARNING: Failed to connect to lvmetad. Falling back to device scanning.
error: cannot seek `/dev/mapper/cryptroot': Invalid argument.
error: cannot seek `/dev/mapper/cryptroot': Invalid argument.
error: cannot seek `/dev/mapper/cryptroot': Invalid argument.
error: cannot seek `/dev/mapper/cryptroot': Invalid argument.
Found linux image: /boot/vmlinuz-4.15.0-29-generic
Found initrd image: /boot/initrd.img-4.15.0-29-generic
Found linux image: /boot/vmlinuz-4.15.0-24-generic
Found initrd image: /boot/initrd.img-4.15.0-24-generic
  WARNING: Failed to connect to lvmetad. Falling back to device scanning.
  WARNING: Failed to connect to lvmetad. Falling back to device scanning.
error: cannot seek `/dev/mapper/cryptroot': Invalid argument.
error: cannot seek `/dev/mapper/cryptroot': Invalid argument.
error: cannot seek `/dev/mapper/cryptroot': Invalid argument.
error: cannot seek `/dev/mapper/cryptroot': Invalid argument.
  WARNING: Failed to connect to lvmetad. Falling back to device scanning.
done
```

These can generally be ignored. They occur because we are actually still inside the rescue system, which does not have the lvmetad service, but it will be available in our actual installation. The errors are caused during creation of the new configuration while running ```update-grub```. If we instead run ```grub-mkconfig```, we are able to see that the files /etc/grub.d/10_linux und /etc/grub.d/20_linux_xen are responsible for these errors. If we want to deactivate these error notices, we would need to remove the following code block from both files:

```bash
# btrfs may reside on multiple devices. We cannot pass them as value of root= parameter
# and mounting btrfs requires user space scanning, so force UUID in this case.
if [ "x${GRUB_DEVICE_UUID}" = "x" ] || [ "x${GRUB_DISABLE_LINUX_UUID}" = "xtrue" ] \
  || ! test -e "/dev/disk/by-uuid/${GRUB_DEVICE_UUID}" \
  || ( test -e "${GRUB_DEVICE}" && uses_abstraction "${GRUB_DEVICE}" lvm ); then
LINUX_ROOT_DEVICE=${GRUB_DEVICE}
else
LINUX_ROOT_DEVICE=UUID=${GRUB_DEVICE_UUID}
fi
```

Afterwards, we can run the command `update-grub` once more.

26\) We will now leave the chroot environment, unmount the directories, and reboot the server:

```bash
exit
umount /mnt/boot /mnt/proc /mnt/sys /mnt/dev
umount /mnt
sync
reboot
```
27\) The server should now boot into dropbear. You will need to connect via SSH, using the correct keypair, and enter your drive encryption key to boot the system:

```bash
ssh -i .ssh/dropbear root@server.example.org
```

You will be greeted with:
```bash
  To unlock root partition, and maybe others like swap, run `cryptroot-unlock`

  BusyBox v1.27.2 (Ubuntu 1:1.27.2-2ubuntu3) built-in shell (ash)
  Enter 'help' for a list of built-in commands.
```

To unlock the system, the command `cryptroot-unlock` must be run and the correct encryption key entered. 

You may see the following output:
```bash
/bin/cryptroot-unlock: line 1: usleep: not found
/bin/cryptroot-unlock: line 1: usleep: not found
/bin/cryptroot-unlock: line 1: usleep: not found
Error: Timeout reached while waiting for PID 388.
```

Regardless, several seconds later the server will terminate the SSH connection and boot the operating system. 

28\) Connect to the server using your normal SSH key pair. This was provided in the Hetzner "Robot" SSH key management at the time installimage was run during the first steps of this guide.
