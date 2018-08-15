---
layout: post
title: Using btrfs with DigitalOcean
featured-img: ocean
---

I'm a long-time customer of Digital Ocean.  It's a great service.  It provides
virtual private servers at an affordable price (as low as $5 a month for 25GB
disk and 1GB RAM), it gives you a permanent IP address, and it has excellent
support.

But there's one thing that I don't like about it. Every Linux droplet image it
provides uses ext4 for its root file system. If you want a different root file
system like btrfs, you're out of luck. In an age when all the standard linux
distributions have moved to file systems that provide modern amenities like
snapshotting, online resizing, and multi-volume balancing, ext4 feels more
than a little antiquated. Moreover, ubiquitous container solutions like docker
can exploit btrfs to improve efficiency and speed, and that's the kind of
goodness I need when I'm running a container-based application like
[nextcloud](https://nextcloud.com/) on a VPS.

So I decided I'd try to figure out how to get a Digital Ocean droplet running
btrfs as its root file system. After some hacking around, I finally got it
working. This post documents the required steps. I should warn you that there
are a lot of steps, and it will probably take you a couple of hours to repeat
them. But after you've done it once, you can save off a snapshot and build
future btrfs droplets from it.  (And hey, DigitalOcean?  Perhaps you should
consider providing btrfs-based images!)

## Creating the droplet

The first step is to visit your account's Droplets page and click the button
to create a new Droplet. When selecting a linux distribution for your droplet,
I recommend Ubuntu 16.04.5. I tried Ubuntu 18.04.1 but found it to be sluggish
on Digital Ocean at the time I wrote this.  If you use a distribution other
than Ubuntu, the instructions in this post may need to be modified somewhat
due to the different tools and filenames used by other distributions.

![Create your droplet]({{ site.baseurl }}/assets/img/posts/btrfs-do/create-droplet.png)

Once you've configured your droplet and given it a name, click the `Create`
button and wait for it to be created.

Now that the droplet is created, select it and go to the `Recovery` tab. After
clicking the `Boot from Recovery ISO` button, toggle your droplet off and on
again. This will cause your new droplet to boot into recovery mode, where
you'll be able to configure it through the access console.

![Switch to recovery]({{ site.baseurl }}/assets/img/posts/btrfs-do/switch-to-recovery.png)

Select the droplet's `Access` tab and click the `Launch Console` button. A new
console window should appear.  After a few minutes, it should look something
like this:

![Recovery console]({{ site.baseurl }}/assets/img/posts/btrfs-do/recovery-console.png)

You might need to hit the return key once or twice to make the menu above
appear.

Type `6` to start the interactive bash shell.  Now the real fun begins.

## Creating the file system

To get a working btrfs root partition, you'll need to do it in a way that
preserves the data in your ext4 file system, but you won't be able to use the
`btrfs-convert` tool. I tried the `btrfs-convert` approach, but I always ended
up with a corrupted btrfs file system. The tool is clearly not ready for prime
time. In fact, I believe it no longer ships with the latest versions of
debian.

To get started, type `parted` at the console's command prompt. This launches
the partition editor tool, which will allow you to manipulate your droplet's
partitions.  You'll be using this tool a lot during this process.

```
$ parted
Warning: Unale to open /dev/sr0 read-write (Read-only file system). /dev/sr0
has been opened read-only.
GNU Parted 3.2
Using /dev/sr0
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted)
```

Type `select /dev/vda` to select the virtual disk device containing your
droplet image.

```
(parted) select /dev/vda
Using /dev/vda
```

Type `print` to view all the partitions on the device.

```
(parted) print
Model: Virtio Block Device (virtblk)
Disk /dev/vda: 26.8GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name  Flags
14      1049kB  5243kB  4194kB                     bios_grub
15      5243kB  116MB   111MB   fat32              msftdata
 1      116MB   26.8GB  26.7GB  ext4
```

The partition you're going to be manipulating is partition number 1, which
contains the droplet's linux image.  The process to replace ext4 with btrfs is
lengthy and consists of the following steps:

  1. Shrink the ext4 file system in partition 1.
  2. Shrink partition 1.
  3. Create a temporary partition 2 in the space you freed up.
  4. Create a file system in temporary partition 2.
  5. Copy the contents of the file system from partition 1 to the file system
     in partition 2.
  6. Delete partition 1.
  7. Create a new partition 1 for the btrfs file system.
  8. Create the btrfs root file system.
  9. Configure subvolumes in the btrfs file system.
  10. Copy the contents of temporary partition 2 into the btrfs file system in
      partition 1.
  11. Delete temporary partition 2.
  12. Resize partition 1 to use nearly all of the disk, leaving space for a
      swap partition.
  13. Create a swap partition.
  14. Fix up `/etc/fstab`.
  15. Reinstall grub.
  16. Rebuild initramfs.
  17. Power off and turn off recovery mode.
  18. Take a snapshot.
  19. Run it!

Whew!  That's a lot of work.  The rest of this post describes each of these
steps in detail.

### Step 1. Shrink the ext4 file system

First you'll need to create a temporary partition to hold the contents of your
original file system. To make room for this new partition, you'll need to
shrink the original partition. But before you can do that, you'll need to
shrink the ext4 file system within the partition.

Whenever you resize a file system, you must first run `e2fsck` on it. So exit
the `parted` tool by typing `quit`, and then type the following at the shell
prompt:

```
$ e2fsck -f /dev/vda1
e2fsck  1.44.1 (24-Mar-2018)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
cloudimg-rootfs: 61265/3225600 files (0.0% non-contiguous), 441790/6525179 blocks
```

Now shrink the ext4 file system to 2GB, which is more than large enough to
hold all of its contents.

```
$ resize2fs /dev/vda1 2G
resize2fs 1.44.1 (24-Mar-2018)
Resizing the filesystem on /dev/vda1 to 524288 (4k) blocks.
The filesystem on /dev/vda1 is now 524288 (4k) blocks long.
```

### Step 2. Shrink the original partition

Now that the ext4 file system is using only the first 2GB of the partition,
you can shrink the partition to make space for another one. Launch the
`parted` tool and `select /dev/vda` again.

```
$ parted
(parted) select /dev/vda
Using /dev/vda
```

Shrink the original partition to 3GB, which is more than enough to hold the
2GB file system.  When asked if you want to continue, type `Yes`.

```
(parted) resizepart 1 3GB
Warning: Shrinking a partition can cause data loss, are you sure you want to
continue?
Yes/No? Yes
```

Make sure the partition was resized correctly by typing `print`.

```
(parted) print
Model: Virtio Block Device (virtblk)
Disk /dev/vda: 26.8GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name  Flags
14      1049kB  5243kB  4194kB                     bios_grub
15      5243kB  116MB   111MB   fat32              msftdata
 1      116MB   3000MB  2884MB  ext4
```

### Step 3. Create a new temporary partition

Now you're going to create a temporary partition into which you can copy the
contents of the original ext4 partition.  In `parted`, type the following
to create a temporary 3GB ext4 partition:

```
(parted) mkpart temp ext4 3GB 6GB
```

Type `print` to make sure the partition was created:

```
(parted) print
Model: Virtio Block Device (virtblk)
Disk /dev/vda: 26.8GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name  Flags
14      1049kB  5243kB  4194kB                     bios_grub
15      5243kB  116MB   111MB   fat32              msftdata
 1      116MB   3000MB  2884MB  ext4
 2      3001MB  6000MB  2999MB  ext4         temp
```

### Step 4. Format the temporary partition's file system

Exit the `parted` tool by typing `quit` and create an ext4 file system
in the temporary partition with the `mkfs.ext4` command:

```
$ mkfs.ext4 /dev/vda2
mke2fs 1.44.1 (24-Mar-2018)
Creating filesystem with 732160 4k blocks and 183264 inodes
Filesystem UUID: 3736c0ed-94d5-421a-8eab-a40f2cfb5b72
Superblock badckups stored on blocks:
        32768, 98304, 163840, 229376, 294912

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
```

### Step 5. Copy the contents of partition 1 to partition 2

Mount the original ext4 file system in partition 1 onto the `/mnt`
directory.

```
$ mount /dev/vda1 /mnt
```

Create a `/mnt2` directory for the file system in temporary partition 2,
and mount it there.

```
$ mkdir /mnt2
$ mount /dev/vda2 /mnt2
```

Type `df -hT` to make sure `/dev/vda1` and `/dev/vda2` were mounted correctly.

```
$ df -hT
Filesystem     Type      Size  Used Avail Use% Mounted on
udev           devtmpfs  479M     0  479M   0% /dev
tmpfs          tmpfs      99M  5.4M   94M   6% /run
/dev/sr0       iso9660   251M  251M     0 100% /lib/live/mount/medium
/dev/loop0     squashfs  220M  220M     0 100% /lib/live/mount/rootfs/rescue_rootfs.squashfs
tmpfs          tmpfs     493M     0  493M   0% /lib/live/mount/overlay
overlay        overlay   493M  7.8M  486M   2% /
tmpfs          tmpfs     493M     0  493M   0% /dev/shm
tmpfs          tmpfs     5.0M     0  5.0M   0% /run/lock
tmpfs          tmpfs     493M     0  493M   0% /sys/fs/cgroup
tmpfs          tmpfs      99M     0   99M   0% /run/user/0
/dev/vda1      ext4      1.9G  871M  1.1G  46% /mnt
/dev/vda2      ext4      2.7G  8.4M  2.6G   1% /mnt2
```

Copy everything from the first partition to the second:

```
$ rsync -aX /mnt/ /mnt2
```

The trailing slash on `/mnt/` is important.

It is now safe to delete the original partition.

### Step 6. Delete the original partition

Unmount partition 1.

```
$ umount /mnt
```

Launch `parted` and remove the first partition.

```
$ parted
(parted) select /dev/vda
Using /dev/vda
(parted) rm 1
(parted) print
Model: Virtio Block Device (virtblk)
Disk /dev/vda: 26.8GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name  Flags
14      1049kB  5243kB  4194kB                     bios_grub
15      5243kB  116MB   111MB   fat32              msftdata
 2      3001MB  6000MB  2999MB  ext4         temp
```

You now have your temporary partition and a gap where the original partition
used to be.  You're going to fill this gap with a new btrfs root partition.

### Step 7. Create the new root partition

Now that the original ext4 partition is gone, create a new partition in its
place to hold the btrfs root file system.

```
(parted) mkpart root btrfs 116M 3GB
(parted) print
Model: Virtio Block Device (virtblk)
Disk /dev/vda: 26.8GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name  Flags
14      1049kB  5243kB  4194kB                     bios_grub
15      5243kB  116MB   111MB   fat32              msftdata
 1       116MB  3001MB  2885MB  btrfs        root
 2      3001MB  6000MB  2999MB  ext4         temp
```

### Step 8. Create the btrfs file system

Type `quit` to exit the `parted` tool and type the following to create the
btrfs file system on partition 1.

```
$ mkfs.btrfs -f /dev/vda1
btrfs-progs v4.15.1
See http://btrfs.wiki.kernel.org for more information.

Label:              (null)
UUID:               59ef3c78-be03-44e2-8a77-2e1b586c9de4
Node size:          16384
Sector size:        4096
Filesystem size:    2.69GiB
Block group profiles:
  Data:             single            8.00MiB
  Metadata:         DUP             137.50MiB
  System:           DUP               8.00MiB
SSD detected:       no
Incompat features:  extref, skinny-metadata
Number of devices:  1
Devices:
   ID        SIZE  PATH
    1     2.69GiB  /dev/vda1
```

### Step 9. Set up btrfs subvolumes

Unlike ext4, btrfs supports subvolumes. I like to configure mine to give me
more flexibility with snapshots. So I usually create a root `@` subvolume, a
`@home` subvolume for home directories, a `@tmp` subvolume for temporary
files, and a `@snapshots` subvolume for btrfs snapshots. If you don't want to
configure your subvolumes the same way I do, you can get away with creating
only the `@` subvolume, but keep in mind that you'll need to return to the
droplet recovery system if you decide to change your subvolume organization
later.

To start creating subvolumes, mount the btrfs file system onto `/mnt`.

```
$ mount /dev/vda1 /mnt -o subvolid=0
```

Now create the subvolumes.

```
$ btrfs subvolume create /mnt/@
Create subvolume '/mnt/@'
$ btrfs subvolume create /mnt/@home
Create subvolume '/mnt/@home'
$ btrfs subvolume create /mnt/@tmp
Create subvolume '/mnt/@tmp'
$ btrfs subvolume create /mnt/@snapshots
Create subvolume '/mnt/@snapshots'
```

Change the permissions of the `@tmp` subvolume since anyone should be able to
read and write it.

```
$ cd /mnt
$ chmod 1777 @tmp
```

Create mount points in the root `@` subvolume for the other subvolumes.

```
$ cd /mnt/@
$ mkdir home tmp snapshots
$ chmod 1777 tmp
```

At this point, feel free to create any other subvolumes you might need.  It
will be more difficult to do so later on.

### Step 10. Copy the temporary partition's file system into the new btrfs file system

Copy everything from the temporary partition into the new root subvolume:

```
$ rsync -aX /mnt2/ /mnt/@
```

The trailing slash on `/mnt2/` is important.

### Step 11. Delete the temporary partition.

Now that everything has been copied into the new root btrfs file system,
you can delete temporary partition 2.

First unmount the temporary file system on partition 2.

```
$ umount /mnt2
```

Then delete the temporary partition using `parted`.

```
$ parted
(parted) select /dev/vda
Using /dev/vda
(parted) rm 2
(parted) print
Model: Virtio Block Device (virtblk)
Disk /dev/vda: 26.8GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name  Flags
14      1049kB  5243kB  4194kB                     bios_grub
15      5243kB   116MB   111MB  fat32              msftdata
 1       116MB  3001MB  2885MB  btrfs        root
```

At this point, your temporary partition should be gone leaving only a small
root btrfs partition and the boot and recovery partitions.

### Step 12. Resize the btrfs partition

Now that you have your root file system in the first partition of the disk,
you'll want to resize the partition to use up most of the remaining space. You
don't want to use up *all* the remaining space, however. The reason is that
you'll want to reserve some of this space for a swap partition. Unlike the
ext4 file system, you cannot use a swap file and `swapon` in btrfs; you must
have a separate swap partition.

My general rule of thumb for sizing swap partitions on systems with small
memory footprints is to multiply the amount of physical memory by two.  If
you're building a droplet with 1GB memory, that means 2GB of swap space. You
can get by with reserving less for your swap partition if you don't plan to
use much memory.

First type `quit` to exit `parted` and unmount the btrfs partition.

```
$ umount /mnt
```

Now run `parted` again and resize partition 1. To calculate the size of the
partition, use the disk size reported by `parted` and subtract the amount of
swap space (plus a small fudge factor) you want to reserve.  In my example,
`parted` reports that my drive has `26.8GB` of space, and I want to reserve
`2GB` for swap, so I set the partition's end value to `24.7GB`.

```
$ parted
(parted) select /dev/vda
Using /dev/vda
(parted) resizepart 1 24.7GB
(parted) print
Model: Virtio Block Device (virtblk)
Disk /dev/vda: 26.8GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name  Flags
14      1049kB  5243kB  4194kB                     bios_grub
15      5243kB  116MB    111MB  fat32              msftdata
 1       116MB 24.7GB   24.6GB  btrfs        root
```

Quit `parted` and mount the btrfs root `@` subvolume onto `/mnt`.

```
$ mount /dev/vda1 /mnt -o defaults,subvol=@
```

Resize the btrfs file system to use all the space in the partition.

```
$ btrfs filesystem resize max /mnt
Resize '/mnt' of 'max'
```

You now have a root btrfs file system using up nearly the entire disk, minus
some space for a swap partition.  Type `df -Th` to verify that it worked.

### Step 13. Create the swap partition

You are now going to create the swap partition. First, examine your partition
table using `parted`.

```
$ parted
(parted) select /dev/vda
Using /dev/vda
(parted) print
Model: Virtio Block Device (virtblk)
Disk /dev/vda: 26.8GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name  Flags
14      1049kB  5243kB  4194kB                     bios_grub
15      5243kB  116MB    111MB  fat32              msftdata
 1       116MB 24.7GB   24.6GB  btrfs        root
```

In my example, I am going to create a swap partition that starts at `24.7GB`
and ends at `26.8GB`.  If you used a different disk size or reserved a
different amount of swap space, you'll need to calculate your start and end
values differently.

Use the `mkpart` command to create the swap partition:

```
(parted) mkpart swap linux-swap 24.7GB 26.8GB
(parted) print
Model: Virtio Block Device (virtblk)
Disk /dev/vda: 26.8GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system     Name  Flags
14      1049kB  5243kB  4194kB                        bios_grub
15      5243kB  116MB    111MB  fat32                 msftdata
 1       116MB 24.7GB   24.6GB  btrfs           root
 2      24.7GB 26.8GB   2142MB  linux-swap(v1)  swap
```

You can now quit `parted`.  You're done with that tool (finally!).

Finally, for linux to use the swap partition correctly, you need to run
`mkswap` on the newly created swap partition.

```
$ mkswap -f /dev/vda2
```

### Step 14. Fix up fstab

Now that you've switched your root file system from ext4 to btrfs, you need
to fix up your `/etc/fstab` file.

Your current `fstab` file should look something like this:

```
$ cat /mnt/etc/fstab
LABEL=cloudimg-rootfs   /       ext4   defaults        0 0
LABEL=UEFI       /boot/efi     vfat    defaults        0 0
```

Using the `vim` editor, modify your `/mnt/etc/fstab` file so it looks like
this:

```
/dev/vda1       /               btrfs   defaults,subvol=@               0       1
/dev/vda1       /tmp            btrfs   defaults,subvol=@tmp            0       2
/dev/vda1       /home           btrfs   defaults,subvol=@home           0       2
/dev/vda1       /snapshots      btrfs   defaults,subvol=@snapshots      0       2
/dev/vda2       none            swap    sw                              0       0
LABEL=UEFI      /boot/efi       vfat    defaults
```

If you didn't use the same subvolumes as I did, you may need to tweak your
`fstab` file a bit.

I formatted my `fstab` file so that it was nicely aligned, but that's not
necessary. You can always go back and fix up your `fstab` later once you're no
longer using the kludgy access console.

### Step 15. Reinstall grub

Because you replaced ext4 with btrfs, you need to reinstall your grub
boot loader.

```
$ mount -t proc proc /mnt/proc
$ mount -t sysfs sys /mnt/sys
$ mount -o bind /dev /mnt/dev
$ chroot /mnt grub-mkconfig -o /boot/grub/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-4.4.0-131-generic
Found initrd image: /boot/initrd.img-4.4.0-131-generic
  /run/lvm/lvmetad.socket: connect failed: No such file or directory
  WARNING: Failed to connect to lvmetad. Falling back to internal scanning.
done
$ chroot /mnt grub-install /dev/vda
Installing for i386-pc platform.
Installation finished. No error reported.
```

Grub is now updated so that your droplet image can boot under btrfs.

### Step 16. Rebuild initramfs

The last step before rebooting is to rebuild initramfs for your linux
image.

The first thing you need to do is figure out which version of the kernel
you're running.  Take a look at the files in the  `/mnt/boot` directory:

```
$ ls /mnt/boot
System.map-4.4.0-131-generic  grub
abi-4.4.0-131-generic         initrd.img-4.4.0-131-generic
config-4.4.0-131-generic      retpoline-4.4.0-131-generic
efi                           vmlinuz-4.4.0-131-generic
```

As you can see in this example, this droplet image is using kernel version
`4.4.0-131-generic`.  Your system may show a different version, so keep track
of it.  You'll need to append the version string to `linux-image-` in the
following step:

```
$ chroot /mnt /usr/sbin/dpkg-reconfigure linux-image-4.4.0-131-generic
Found kernel: /boot/vmlinuz-4.4.0-131-generic
Updating /boot/grub/menu.lst ... done

run-parts: executing /etc/kernel/postinst.d/zz-update-grub 4.4.0-131-generic
  /boot/vmlinuz-4.4.0-131-generic
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-4.4.0-131-generic
Found initrd image: /boot/initrd.img-4.4.0-131.generic
  /run/lvm/lvmetad.socket: connect failed: No such file or directory
  WARNING: Failed to connect to lvmetad. Falling back to internal scanning.
done
```

If a screen pops up asking what to do about the `/boot/grub/menu.lst` file,
select the default option of "Keep the local version currently installed".

### Step 17. Power off and turn off recovery mode.

Everything is ready to go.  Power off the droplet by typing the following:

```
$ poweroff
```

Then return to the droplet `Recovery` tab, switch the droplet to `Off`
(if it's not already shown as off), and select `Boot from Hard Drive`.
Leave the droplet off, because there's one final step you're going to want
to take.

### Step 18. Take a snapshot

Go to the droplet's `Snapshots` tab, and take a snapshot of your new
btrfs-based droplet.  You'll be able to use this snapshot later to build
future btrfs-based droplets without having to go through this arduous process
again. The snapshot should be under 1GB, so it will only cost you 5 cents a
month to keep it around.

### Step 19. Run it!

Switch the droplet to `On`, and you should be running btrfs as your
root file system!

## Cool things you can do

Once you're running btrfs as the root file system of your droplet, you can add
a volume to your droplet and painlessly grow (or shrink) the size of your root
file system whenever you like.  For example, I added a 10GB block volume to
one of my droplets (at a price of $1 per month), and then ran the following
btrfs commands from my droplet:

```
$ btrfs device add /dev/disk/by-id/scsi-0DO-Volume_butter-10gb /
$ btrfs filesystem resize max /
$ btrfs filesystem balance /
```

It's awesome to be able to add new storage capacity to my root file system
whenever I like.