`zfsbootmenu-installer`
=======================

* First-class Root-on-ZFS
* Secure Boot
* Boot Environments

`zfsbootmenu-installer` is a shell script for installing Root-on-ZFS
setups for a small handful of GNU/Linux distributions.
Download a Live CD and boot your new machine to it, connect to the
Internet, then download this installer and run it to install a
supported GNU/Linux distribution with Root-on-ZFS, Secure Boot,
and Boot Environments.

`zfsbootmenu-installer` can set up a ZFS pool on a single disk, but
it can also set up arrays into ZFS VDEVs of `mirror`, `raidz1`,
`raidz2`, or `raidz3`.

`zfsbootmenu-installer` supports "first-class" Root-on-ZFS; unlike
with the Grub bootloader, where Grub has only the barest ability to
read ZFS Pools with older features, it installs
[ZFSBootMenu](https://zfsbootmenu.org/), a specialized
Root-on-ZFS bootloader, into the storage devices in your array.
Not only can ZFSBootMenu read the latest features of ZFS, it can
also write to the ZFS Pool: for example, it can rollback a bad
upgrade of your operating system.

`zfsbootmenu-installer` also has Secure Boot support, by using a
Microsoft-signed [Fedora Shim](https://github.com/rhboot/shim)
bootloader as first stage to start your computer, setting up
Shim to load ZFSBootMenu as second stage.
It automates creation of a Machine Owner Key (MOK, a feature of
Shim that allows you to provide your own key without re-flashing
your motherboard's firmware or searching through its menus for a
way to add your own key) in order to sign ZFSBootMenu itself, as
well as your kernels and kernel modules, including, if needed, the
OpenZFS module.
It even sets up the installed distribution so that new kernels and
DKMS modules will be signed by the Machine Owner Key.

The advantage of ZFSBootMenu is that in combination with snapshots,
you no longer have to fear that updating your operating system
will prevent you from booting up your computer.
Even if an update manages to break your system enough that the
entire OS won't boot, ZFSBootMenu is capable of rolling back the
OS's ZFS dataset to an older snapshot, hopefully one predating
whatever it is that broke your OS.
In fact, ZFSBootMenu is itself a (small) Linux OS all by itself,
and can drop into a recovery shell that lets you emergency-manage
your ZFS Pool.

Your `/home` directory has a separate ZFS dataset from the
installed operating system, so that rolling back your operating
system does not also rollback your data (and vice-versa).

Another advantage of ZFSBootMenu is that it supports
[Boot Environments](https://docs.zfsbootmenu.org/en/v3.1.x/general/bootenvs-and-you.html).
A Boot Environment is simply a complete GNU/Linux operating
system you can boot, stored in its own ZFS dataset, independent
of other operating systems on their own ZFS dataset.
The first distribution you install with `zfsbootmenu-installer`
is simply your first Boot Environment.

Boot Environments enable flexible multi-booting of different
GNU/Linux operating systems, without requiring you to partition
your hard disk (a risky operation) --- ZFS datasets *are* the
partitions between OSs!

Having the Boot Environments feature mean you can install an
alternate distribution, or a later major version of the same
distribution, and then try it out, by simply creating a new ZFS
dataset and installing the new distribution or version there
--- it now becomes just another Boot Environment you can boot
from, while still being able to access your beloved `/home`
directory.
`zfsbootmenu-installer` also includes the ability to add one
of the Linux distributions it supports as a new Boot Environment,
automatically creating the ZFS dataset and installing the OS
for you.
If the new distribution or version works well for you, you
can switch to using that Boot Environment as the default one
(`zfsbootmenu-installer` lets you change the `bootfs` option
on your ZFS pool to do so) and later destroy your previous Boot
Environment to free up space.
Or if the new distribution or version is not working well for
you, you can just destroy that alternate Boot Environment and
retain your current one.

Opinionated
-----------

`zfsbootmenu-installer` has the very strong opinion that you
must encrypt your ZFS pool, and will not disallow this.
To compensate, it sets up installed distributions to
automatically log you in, so that you only need enter your
password once, to unlock the encryption.
The encryption key *is* the password.

(Note that the password entry is "blind" i.e. it will not
show asterisks or circles for each character you type.
Just keep typing the entire password and trust it is
being checked.)

Encryption is vital to protecting your Machine Owner Key
while the machine is turned off, as the Machine Owner Key
is stored passphrase-free on your ZFS Pool in order to
automate signing of new kernels and kernel modules; if the
Machine Owner Key is subverted, then in practice Secure Boot
is subverted, which is why `zfsbootmenu-installer` insists
on encryption.

As the "log you in" above suggests, this also has the strong
opnion that there is only one true controller and owner of
the machine and that any other user will share that owner's
username, with access control being physical access to the
computer.
This is the usual practice for homelab and personal / family
computers anyway.

`zfsbootmenu-installer` installs an EFI System Partition to
all disks in the array it initializes.
This lets you boot even if any arbitrary disk breaks; it does
not matter which one breaks, any of them can still be booted
(you may need to change boot priorities in your motherboard
firmware interface).
If you need to add or replace disks, `zfsbootmenu-installer`
can format it with an EFI System Partition before you
manually use `zpool add`, `zpool attach`, or `zpool replace`
on the partition on the rest of the disk, to retain this
ability to boot your system regardless of whichever disk
breaks.

You *must* still use ZFS even if you don't have ECC RAM.
Remember, lesser filesystems will still break whether or
not you have ECC RAM; at least with ZFS, the probability of
the filesystem experiencing bitrot is much lower, even
without ECC RAM, than using any other filesystem without
ECC RAM.
Think of ECC RAM as simply closing the last loophole for
data loss; ZFS closes all the others that lesser filesystems
won't.

You *must* still use ZFS even if it's a single device.
Remember, whole-device failure is the biggest enemy but it's
not the ***only*** enemy.
Random bitflips can happen, and you need a full integrity
system to validate your precious data on that one disk you
have, and ZFS gets that for you.
And snapshots, come on!
Hacking your OS to add drivers to support that expensive
hardware?
Yes, that's totally safe and not going to risk making your
system unbootable, but in the off chance it actually does,
you can roll back to an older snapshot.
Sure, TEMU ZFS I mean BTRFS has integrity and snapshots
too, but it also has the literally killer feature of
suddenly collapsing if you make a mistake (I put my `/home`
partition into a BTRFS `raid5`.  Once.), just use ZFS
already.

`zfsbootmenu-installer` does not set up a swap partition.
Ideally, the swap partition would be its own `zvol` on the
same ZFS Pool you are using.
Unfortunately, this is not safe, since there is an
unfixable deadlock condition when swapping on a ZFS (there
are guides on the Internet about how to make `zvol` swap
partitions, but they only mitigate the risk of deadlock,
not remove it completely).
The issue is figuring out how to set up a swap partition
when doing a multi-disk array; do we reserve a swap
partition on each disk?
How do we select the swap partition to use, if the disk
whose swap partition we happen to be using goes down?
(The ZFS Pool will continue operating, but what about
the swap partition setup?)
How big a swap space?

Usage
-----

Download a Live CD, preferably Ubuntu's so that you don't
need to install ZFS, and because Ubuntu's Live CD is more
likely to be bootable on a Secure Boot machine.

Ensure the Live CD has ZFS support (install `zfs-dkms`
and `zfsutils-linux` on a Debian Live environment, for
example), connect to the Internet, download this script,
make it executable, and run it on a terminal as the
superuser.

Then select the `Initialize` option, provide your preferred
hostname, username, password, then configure the ZFS Pool
and select the first GNU/Linux distro to install, then
sit back and wait (the installation process is likely to
ask you things like timezones, keyboard configurations, etc).

Once the installation is finished, you can reboot the
machine (remember to remove the Live CD USB).
Once rebooted, you will enter "MOK Management".
Select "Enrol MOK", then "Continue" and enter your password
to register your Machine Owner Key for your new machine.
Then the machine will restart rebooting with the Machine
Owner Key registered, and Secure Boot capable.

Once you boot up into ZFSBootMenu and select your first
Boot Environment, you can enter your password to unlock
the ZFS Pool encryption and start using your machine
with Root-on-ZFS, Secure Boot, rollbackable snapshots,
and the ability to install other supported GNU/Linux
distros.

`zfsbootmenu-installer` will install itself into your
new system so that you can use it to install more
GNU/Linux distros or prepare new disks to add or replace
into the ZFS array.
Remember it needs to be run as superuser.

Manually Installing New Boot Environments
-----------------------------------------

There are far too many GNU/Linux distributions out there,
and it is simply not possible for `zfsbootmenu-installer`
to support each one.

If you have installed a working system with
`zfsbootmenu-installer` and you are hankering for a new
Boot Environment for some random distribution that it
doesn't support, you'll need:

* Some way to "bootstrap into a `chroot` jail" for
  that distribution.
  For example:
  - `debootstrap` for most Debian derivatives.
  - `pacstrap` for Arch Linux.
  - `dnf --installroot=` and some hacking for Red
    Hat derivatives.
* Some way to get ZFS installed in that distribution.
  - The worst-case for many distributions is to
    directly compile packages of OpenZFS yourself
    from within that distribution's `chroot` jail,
    see:
    https://openzfs.github.io/openzfs-docs/Developer%20Resources/Custom%20Packages.html

First, create the ZFS dataset and mount it somewhere
(you have to do this as superuser):

    mkdir -p /work
    zfs create -o mountpoint=/work -o canmount=noauto zroot/ROOT/DistroName
    zfs mount zroot/ROOT/DistroName

Then use the "bootstrap into `chroot` jail" process
for your distro into the `/work` dir.
For example, for Debian 12:

    debootstrap bookworm /work

You will need to copy a few things from your current
running distro to the new distro:

    cp /etc/hostid /work/etc/hostid
    cp /etc/hostname /work/etc/hostname
    cp /etc/hosts /work/etc/hosts
    mkdir -p /work/etc/zfs
    cp /etc/zfs/zroot.key /work/etc/zfs
    mkdir -p /work/usr/local/bin
    cp /usr/local/bin/zfsbootmenu-installer /work/usr/local/bin
    cp -R /usr/local/mok /work/usr/local
    cp /usr/local/username /work/usr/local

Then you'll need to start installing.
Generally for `chroot` jails like this, you want to
mount some things in standard places before you can
actually safely `chroot` into it:

    mount -t proc proc /work/proc
    mount -t sysfs sys /work/sys
    mount -B /dev/ /work/dev
    mount -t devpts pts /work/dev/pts

After that you can `chroot /work /bin/bash` to
enter the `chroot` jail.

At this point, you usually have to set up your
package sources if the bootstrap process doesn't
do that for you (`debootstrap` doesn't, for
instance, you have to manually create an initial
`/etc/apt/sources.list`).
Do that now if it's needed for your bootstrap
process.

***But you can't start installing anything yet!***
You also need to automate signing kernels and
kernel modules with your MOK or you still can't
boot in Secure Boot.
And you have to set up that signing before you
install anything so that you auto-sign everything
installed.

If the distro uses a recent-enough DKMS, and the
ZFS kernel module on it uses DKMS (or if you will
use a manually-built DKMS package build of OpenZFS),
you can usually do something like:

    mkdir -p /etc/dkms/framework.conf.d/
    cat > /etc/dkms/framework.conf.d/00-0-sign-with-mok <<SCRIPT
    mok_signing_key=/usr/local/mok/MOK.priv
    mok_certificate=/usr/local/mok/MOK.der
    SCRIPT

Because you are relying on DKMS to automate signing
of kernel modules, you should avoid precompiled,
non-DKMS kernel modules from your distro, unless
you know how to automate signing them with your own
MOK (I don't, I can only find automation for DKMS
kernel modules).

For kernel signing, you need the distribution to
have `sbsign` somewhere in its packaging --- on
most Debian derivatives, it's in a package called
`sbsigntool`.
Install it, and only it, now (e.g. use
`--no-install-recommends` with `apt` just to make
sure you don't install anything else yet).
Then you need to set up auto-signing of installed
kernels.

On Debian-derived systems, the below would usually
work:

    mkdir -p /etc/kernel/postinst.d/
    cat > /etc/kernel/postinst.d/00-0-sign-with-mok <<SCRIPT
    #! /bin/sh
    KERNEL="$2"
    exec sbsign --key /usr/local/mok/MOK.priv --cert /usr/local/mok/MOK.pem "$KERNEL" --output "$KERNEL"
    SCRIPT
    chmod +x /etc/kernel/postinst.d/00-0-sign-with-mok

This is somewhat more hard to find automation for
online (searching "automatic MOK signing kernel
$DISTRO" will usually lead you to instructions on
how to automatically sign kernel ***modules***,
not kernels).
Try looking for post-install hooks for kernels
instead.

To check that the kernels *are* signed, use
`sbverify --list $KERNEL` to see if the kernels
are also signed with your Machine Owner Key.

In the worst case, you can look for post-install
hooks for the package manager of the distribution,
which trigger whenever any package is installed.
Then you can install a post-install hook that
invokes the following command as superuser:

    /usr/local/bin/zfsbootmenu-installer sign-all-kernels

There's also two more things you want to somehow
invoke when your package manager changes your
system:

* `/usr/local/bin/zfsbootmenu-installer zbmi-post-new-hook`
  - Invoke this every time the package manager adds a new
    file to the system, or overwrites a file with new
    contents.
* `/usr/local/bin/zfsbootmenu-installer zbmi-post-change-hook`
  - Invoke this every time the package manager adds, modifies,
    or removes files from the system.

On a Debian derivative that uses `dpkg`/`apt`,
the below installs both hooks into `dpkg`:

    /usr/local/bin/zfsbootmenu-installer debian-derivative-install-zbmi-hooks

Once you have automatic signing of kernels and
kernel modules, and hooked the two calls above,
you can start installing the rest of the system
you want to boot.

At minimum you need a Linux kernel, and
you will need to also install ZFS for that Linux
kernel (if the distro refuses to support ZFS for
it, go download directly OpenZFS and compile
packages for the package manager of your distro,
or figure out some way to get it into DKMS).
You also need to set up a user (make sure it is
UID 1000; `zfsbootmenu-installer` makes sure
the username you give is always UID 1000 on all
systems it installs itself) and give that user
administrator capabilities (e.g. add to `sudo`
group or `sudoers` file), or create a password
for root.

You'll need to manually enable automation for
ZFS, too.
On `systemd`-based systems where you installed
some OpenZFS package, you can generally just
do:

    systemctl enable zfs.target
    systemctl enable zfs-import-cache
    systemctl enable zfs-mount
    systemctl enable zfs-import.target
    systemctl enable zfs-volume-wait
    systemctl enable zfs-volumes.target
    systemctl enable zfs-scrub-monthly@zroot.timer
    systemctl enable zfs-trim-monthly@zroot.timer

If the distro uses anything other than `systemd`
for its Init system, then you're probably on your
own to try to figure out how ZFS does its magic
under the hood, heh.
You generally need to at least set up the following
in your Init system:

* Run `modprobe zfs` as early as possible.
* Run `zpool import -a` after `modprobe zfs`.
  - Properly speaking you should be using
    `zpool import -c /etc/zfs/zpool.cache` but
    this can work as a stopgap if that seems
    too brittle in your system.
* Run `zvol_wait`after the above `zpool import -a`.
  Make sure it runs before anything that might
  want `zvol`s.
* Run `zfs mount -a` after the above
  `zpool import -a`.
  This is vital in order to get `/home` mounted
  as early as you can.
  - If the distro supports separate `/home`
    partitions, you can try to look at how the
    distro handles the timing of mounting
    `/home`.
  - Make sure this completes before anything that
    wants a `/home`!
* You also want to schedule regular `zpool scrub`
  and `zpool trim` on your `crontab` system.

Further, you really want `zfs-auto-snapshot`.
On Debian derivatives, this exists as a package
that is nice enough that it automatically sets up
`crontab` entries on installation, but the actual
source code has no automatic set up for the
`crontab` system, so other packagers might do the
bare minimum and just put the scripts into the
package without setting up `crontab` entries.
In particular, the source for `zfs-auto-snapshot`
has no `systemd` `.timer` files.
Yes, ZFS's magic is implemented from actual bits
and bytes.

If the distro has no `zfs-auto-snapshot` package,
you'll need to both download it from the
github:

* https://github.com/zfsonlinux/zfs-auto-snapshot

...and also set up the `crontab` entries manually.

Then once you've installed everything to your
satisfaction for the new system, you can do a
final signing of kernels, since post-install
hooks might not trigger within the `chroot`
jail (DKMS usually does trigger):

    /usr/local/bin/zfsbootmenu-installer sign-all-kernels

And then you can:

    exit

Once you have exited the `chroot` jail, you can
set up the new root dataset for boot:

    umount -qnR /work
    rmdir /work
    zfs set mountpoint=/ zroot/ROOT/DistroName

Then try booting into it and see if it works!

If it doesn't, you can boot one of the other,
known-working Boot Environments and then use
`zfsbootmenu-installer`'s
`ChrootIntoBootEnvironment` to get a shell
into that Boot Environment.
