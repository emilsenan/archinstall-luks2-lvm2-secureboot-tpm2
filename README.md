# Arch Linux install with full disk encryption with LUKS2 ,Logical Volumes with LVM2, Secure Boot and TPM2 Setup
A complete Arch Linux installation guide with **LUKS2** full disk encryption, and logical volumes with **LVM2**, and added security using **Secure Boot** with **Unified Kernel Image** and **TPM2 LUKS** key enrollment for auto unlocking encrypted root.

Firstly, Acquire an installation image. Visit the Download page and, acquire the ISO file and flash it to a USB drive and boot off it.
This guide assumes that your system supports UEFI.

### 1. Disk Preparation

We'll use a 512MB FAT32 system partition for our **EFI** partition , and for the root we'll use an **ext4** partition and a **SWAP** partition using **LVM2** logical volumes inside a LUKS encrypted partition. 

### 2. Partition the disks

We're gonna be using `cfdisk` for partitioning the disks.

Before partitioning , the output of `lsblk` is gonna look something like this.

```
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
nvme0n1     259:0    0  709.5G  0 disk
```

1. Launch `cfdisk`:
Open a terminal. Identify your disk. For this guide, we'll use /dev/nvme0n1 as an example. Replace it with your actual disk identifier.

   ```
   cfdisk /dev/nvme0n1
   ```

3. Select the Label Type:

    Choose **gpt** (GUID Partition Table) if prompted.

4. Create Partitions:

    Create EFI System Partition:
        **Select [ New ]**.
        Enter **512M** for the size.
        Select **[ Type ]** and choose **EFI** System.

    Create LUKS Partition:
        Select **[ New ]**.
        Use the remaining disk space for this partition, or allocate the space you want , if you don't plan on using the entire disk for this setup.
        Ensure the type is Linux filesystem.

    Write Changes:
    
        Select [ Write ].
        Type yes to confirm.

5. Visual Representation of Partition Structure:
    ```
    +------------------+-----------------------+------------+---------------+
    | Partition Number | Partition Type        | Size       | Description   |
    +------------------+-----------------------+------------+---------------+
    | /dev/nvme0n1p1   | EFI System            | 512M       | EFI Partition |
    | /dev/nvme0n1p2   | Linux filesystem      | Remaining  | LUKS2 Volume  |
    +------------------+-----------------------+------------+---------------+
    ```

After partitioning, `lsblk` will output the following.

```
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
nvme0n1     259:0    0  238.5G  0 disk
├─nvme0n1p1 259:1    0   512M  0 part 
└─nvme0n1p2 259:2    0   709G  0 part
```

### 3. Create the encrypted LUKS2 container.

Now we, need to create the **LUKS2** encrypted container.

Create the LUKS encrypted container at the designated partition. Enter the chosen password twice. 

```
# cryptsetup luksFormat /dev/nvme0n1p2
```

Open the container: 

```
# cryptsetup open /dev/nvme0n1p2 cryptlvm
```

Here `cryptlvm` is the name we are assigning to the encrypted container after opening it.

The decrypted container is now available at `/dev/mapper/cryptlvm`.

### 4. Preparing the logical volumes

Create a physical volume on top of the opened LUKS container:

```
# pvcreate /dev/mapper/cryptlvm
```

Create a volume group (in this example, it is named `MyVolGroup`, but it can be whatever you want) and add the previously created physical volume to it: 

```
# vgcreate MyVolGroup /dev/mapper/cryptlvm
```

Create all your logical volumes on the volume group: 

Tip: If a logical volume will be formatted with ext4, leave at least 256 MiB free space in the volume group to allow using `e2scrub`. After creating the last volume with `-l 100%FREE`, this can be accomplished by reducing its size with `lvreduce -L -256M MyVolGroup/home`.

```
# lvcreate -L 4G MyVolGroup -n swap
# lvcreate -L 32G MyVolGroup -n root
# lvcreate -l 100%FREE MyVolGroup -n home
# lvreduce -L -256M MyVolGroup/home
```

Format your file systems on each logical volume:

```
# mkfs.ext4 /dev/MyVolGroup/root
# mkfs.ext4 /dev/MyVolGroup/home
# mkswap /dev/MyVolGroup/swap
```

Mount your file systems:
```
# mount /dev/MyVolGroup/root /mnt
# mount --mkdir /dev/MyVolGroup/home /mnt/home
# swapon /dev/MyVolGroup/swap
```

### 5. Preparing the boot partition

```
# mkfs.fat -F32 /dev/nvme0n1p1
```

Replace `nvme0n1p1` with the drive identifier for your EFI partition.

Mount the partition to `/mnt/efi`:

```
# mount --mkdir /dev/nvme0n1p1 /mnt/efi
```

### 6. Installation

Install essential packages:

```
# pacstrap /mnt base linux linux-firmware linux-headers intel-ucode vim nano efibootmgr sudo
```

You can replace `intel-ucode` with `amd-ucode` if your CPU is an **AMD** CPU

After that is completed, we need to generate the fstab file:

```
# genfstab -U /mnt >> /mnt/etc/fstab
```

Change root into the new system:

```
# arch-chroot /mnt
```

Set the time zone:

```
# ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
```

Replace `Region` and `City` with your corresponding ones.

Run `hwclock` to generate `/etc/adjtime`:

```
# hwclock --systohc
```
This command assumes the hardware clock is set to UTC.

Localization:

Edit `/etc/locale.gen` and uncomment `en_US.UTF-8 UTF-8` and other needed `UTF-8` locales. Generate the locales by running: 

```
# locale-gen
```

Create the `locale.conf` file, and set the LANG variable accordingly:

```
/etc/locale.conf
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
LANG=en_US.UTF-8
```

If you set the console keyboard layout, make the changes persistent in vconsole.conf:

```
/etc/vconsole.conf
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
KEYMAP=de-latin1
```

Network configuration:

Create the `/etc/hostname` file:

```
/etc/hostname
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
yourhostname
```

Edit the hosts file:

Add the following lines:

```
# Static table lookup for hostnames.
# See hosts(5) for details.
127.0.0.1	localhost
::1		localhost
127.0.1.1	yourhostname.localdomain	yourhostname
```

Set the root password:

```
# passwd
```

Create a non-root user account:

```
useradd -m newuser
```

Set the newuser password:

```
# passwd newuser
```

Edit the `/etc/sudoers` file:

Run `visudo`:

Uncomment the following line:

```
%wheel ALL=(ALL:ALL) ALL
```

Add new user to wheel group:

```
# usermod -G wheel newuser
```

### 7. Configuring mkinitcpio

To build a working systemd based initramfs, modify the `HOOKS=` line in mkinitcpio.conf as follows: 
Add the following hooks: **systemd, keyboard, sd-vconsole, sd-encrypt, lvm2**

```
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck)
```

You can skip `sd-vconsole` , if you didn't configure `/etc/vconsole.conf`
Do **not** regenerate the initramfs **yet**, as the `/efi/EFI/Linux` directory needs to be created first , which we will do later


### 8. Configure mkinitcpio for Unified kernel images.

mkinitcpio supports reading kernel parameters from command line files in the `/etc/cmdline.d` directory. Mkinitcpio will concatenate the contents of all files with a `.conf` extension in this directory and use them to generate the kernel command line. Any lines in the command line file that start with a # character are treated as comments and ignored by mkinitcpio. 

Create the `cmdline.d` directory:

```
# mkdir /etc/cmdline.d
```

In order to unlock the encrypted root partition at boot, the following kernel parameters need to be set:

```
/etc/cmdline.d/root.conf
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
rd.luks.name=device-UUID=cryptlvm root=/dev/MyVolGroup/root
```

You can obtain the `device-UUID` by running command `blkid`.

This is an example output.

```
/dev/nvme0n1p1: UUID="1234-5678" BLOCK_SIZE="512" TYPE="vfat" PARTLABEL="EFI System" PARTUUID="11111111-2222-3333-4444-555555555555"
/dev/nvme0n1p2: UUID="abcdef12-3456-7890-abcd-ef1234567890" BLOCK_SIZE="4096" TYPE="crypto_LUKS" PARTLABEL="LUKS" PARTUUID="66666666-7777-8888-9999-000000000000"
/dev/mapper/cryptlvm: UUID="12345678-90ab-cdef-1234-567890abcdef" TYPE="LVM2_member"
/dev/mapper/vg-root: UUID="abcd1234-ef56-7890-abcd-ef1234567890" BLOCK_SIZE="4096" TYPE="ext4"
/dev/mapper/vg-swap: UUID="5678abcd-90ef-1234-5678-90abcdef1234" TYPE="swap"
```

Now you need to obtain the **UUID** for the luks container , in our case for `/dev/nvme0n1p2` which is `abcdef12-3456-7890-abcd-ef1234567890`

Next, modify `/etc/mkinitcpio.d/linux.preset`, as follows, with the appropriate mount point of the EFI system partition: 

Here is a working example linux.preset for the linux kernel and the Arch splash screen. 

```
/etc/mkinitcpio.d/linux.preset
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
# mkinitcpio preset file for the 'linux' package

#ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux"

PRESETS=('default' 'fallback')

#default_config="/etc/mkinitcpio.conf"
#default_image="/boot/initramfs-linux.img"
default_uki="/efi/EFI/Linux/arch-linux.efi"
default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"

#fallback_config="/etc/mkinitcpio.conf"
#fallback_image="/boot/initramfs-linux-fallback.img"
fallback_uki="/efi/EFI/Linux/arch-linux-fallback.efi"
fallback_options="-S autodetect"
```

Finally, to build the **UKI**, make sure that the directory for the UKIs exist.
For example, for the linux preset: 
```
# mkdir -p /efi/EFI/Linux
```

Now install the `lvm2` package:

```
pacman -S lvm2
```

Now, regenerate `initramfs`:

```
# mkinitcpio -p linux
```

Optionally, remove any leftover `initramfs-*.img` from `/boot` or `/efi`. 

### 9. Configuring the boot loader

Install `systemd-boot` with:

```
# bootctl install
```

The Unified kernel image generated by mkinitcpio will be automatically recognized and does not need an entry in `/efi/loader/entries/`.

### 10. Installing NetworkManager

Install NetworkManager to ensure we have network connectivity when we boot into our system later:

```
pacman -S networkmanager && systemctl enable NetworkManager
```

### 11. Reboot into `UEFI`

Now reboot into `UEFI` and put secure boot into **SETUP MODE**. Refer to your motherboard manufaturer's guide on how to do that.

For most systems, you can do this by, just going into **BOOT** tab, **enabling secure boot**, go to **SECURITY** tab and do **Erase all secure boot settings**.

Now save changes and exit.

Now when booting into **Arch Linux** you'll be prompted to enter the passphrase to your LUKS partition.

Enter it and boot into the system. Login as **root**.

### 12. Secure Boot

Now to configure secure boot , first install the `sbctl` utility:

```
$ pacman -S sbctl
```

**Note: It might say completed installation with some errors, that's fine because sbctl can't find the key database, because there never was one.

Now run ```sbctl status``` and ensure setup mode is enabled.

Then create your secure boot keys with:

```
$ sbctl create-keys
```

Enroll the keys, with Microsoft's keys, to the UEFI:

```
$ sbctl enroll-keys -m
```

Warning: Some firmware is signed and verified with Microsoft's keys when secure boot is enabled. Not validating devices could brick them. To enroll your keys without enrolling Microsoft's, run: `sbctl enroll-keys`. Only do this if you know what you are doing.

Check the secure boot status again:

```
$ sbctl status
```

sbctl should be installed now, but secure boot will not work until the boot files have been signed with the keys you just created. 

Check what files need to be signed for secure boot to work:

```
# sbctl verify
```

Now sign all the unsigned files. Most probably these are the files you need to sign:

```
/efi/EFI/BOOT/BOOTX64.EFI
/efi/EFI/Linux/arch-linux-fallback.efi
/efi/EFI/Linux/arch-linux.efi
/efi/EFI/systemd/systemd-bootx64.efi
```

The files that need to be signed will depend on your system's layout, kernel and boot loader. 

```
$ sbctl sign --save /efi/EFI/BOOT/BOOTX64.EFI
$ sbctl sign --save /efi/EFI/Linux/arch-linux-fallback.efi
$ sbctl sign --save /efi/EFI/Linux/arch-linux.efi
$ sbctl sign --save /efi/EFI/systemd/systemd-bootx64.efi
```

The `--save` flag is used to add a pacman hook to automatically sign all new files whenever the **Linux kernel**, **systemd** or the **boot loader** is updated.

Now reboot, and verify that Secure Boot is enabled by using command `bootctl`

```
bootctl
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
System:
      Firmware: UEFI
 Firmware Arch: x64
   Secure Boot: enabled (user)
  TPM2 Support: yes
  Measured UKI: yes
  Boot into FW: supported
```

### 13. Enrolling the TPM

Make sure Secure Boot is active and in user mode when binding to PCR 7, otherwise, unauthorized boot devices could unlock the encrypted volume.
The state of PCR 7 can change if firmware certificates change, which can risk locking the user out. This can be implicitly done by fwupd or explicitly by rotating Secure Boot keys.

To begin, run the following command to list your installed TPMs and the driver in use:

```
$ systemd-cryptenroll --tpm2-device=list
```

First, let's generate a recovery key in case it all gets messed up some time in the future:

```
$ sudo systemd-cryptenroll /dev/nvme0n1p2 --recovery-key
```

Save or write down the recovery key in some safe and secure place.

To check that the new recovery key was enrolled, dump the LUKS configuration and look for a systemd-tpm2 token entry, as well as an additional entry in the Keyslots section: 

```
$ cryptsetup luksDump /dev/nvme0n1p2
```

It will most probably be in `keyslot 1`.

We'll now enroll our system firmware and secure boot state.
This would allow our TPM to unlock our encrypted drive, as long as the state hasn't changed.

```
$ sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+7  /dev/nvme0n1p2
```

If all is well, reboot , and you won't be prompted for a passphrase, unless secure boot is disabled or secure boot state has changed.

### 14. Tips

Now if at some point later in time, our secure boot state has changed, for example by, booting an UBUNTU ISO which adds it's own secure boot keys, the TPM won't unlock our encrypted drive anymore.
To fix it, first enter UEFI, and clear the TPM.

Then boot into Arch Linux, as root.

Then we need to kill keyslots previously used by the **TPM**.

Remove TPM Keyslot:

Figure out which keyslot is being used by the tpm by runnging `cryptsetup luksDump /dev/nvme0n1p2`.

In the **Tokens**: section, look for systemd-tmp2, and under it find the keyslot used:

```
Tokens:
  0: systemd-recovery
	Keyslot:    1
  1: systemd-tpm2
  ...
  ...
	Keyslot:    2
```
As , you can see keyslot **1** is used by `systemd-recovery` and **2** is used by `systemd-tpm2`

Now to kill it run:

```
$ sudo cryptsetup luksKillSlot /dev/nvme0n1p2 2
```

After killing the keyslot, we need to remove the Token associated with it.

```
$ sudo cryptsetup token remove --token-id 1 /dev/nvme0n1p2
```
Here we specify `token-id` as `1` based on the previous output of `luksDump`. Specify it correspondingy depending on what the token number is on your output of `luksDump`.

Now repeat the steps from [TPM enrollment](https://github.com/joelmathewthomas/archinstall-luks2-lvm2-secureboot-tpm2?tab=readme-ov-file#12-enrolling-the-tpm) to renroll to the TPM.


With this, the guide has mostly covered on how to install Arch Linux, Encrypt disk with LUKS2 , use logical volumes with LVM2, how to setup Secure Boot, and how to enroll the TPM.

The only steps remaining are to install a Desktop Environment or a Window Manager, which this guide, unfortunately, will not cover.

### Author

This guide was written by [Joel Mathew Thomas](https://github.com/joelmathewthomas)

### References

For more detailed information, please refer to the [Arch Wiki on Encrypting an Entire System](https://wiki.archlinux.org/title/dm-crypt/Encrypting_an_entire_system).

Thank You.

