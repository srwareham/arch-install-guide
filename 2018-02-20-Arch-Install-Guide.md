Arch Linux Install Guide
=======

This guide is meant to supplement the [official documentation]( https://wiki.archlinux.org/index.php/installation_guide) by demonstrating a start-to-finish system setup.

![](assets/arch-logo-180x180.png)

Although all options are easily configurable, the final default system will:
* Boot using the [GRUB](https://wiki.archlinux.org/index.php/GRUB) bootloader with
 * A [GPT](https://wiki.archlinux.org/index.php/Partitioning#GUID_Partition_Table)  [EFI](https://wiki.archlinux.org/index.php/EFI_System_Partition) partition for [UEFI](https://wiki.archlinux.org/index.php/Unified_Extensible_Firmware_Interface) booting
 * [Intel microcode](https://wiki.archlinux.org/index.php/Microcode#Enabling_Intel_microcode_updates) enabled
* Use a single [BTRFS](https://wiki.archlinux.org/index.php/Btrfs) partition for root/home data storage with [snapper](https://wiki.archlinux.org/index.php/Snapper) for automatic backups and simple full-system restores
* Have one admin user configured with sudo access, zsh, and trizen
* Have pre-configured ethernet networking

**Table of Contents**

[TOC]

Creating a Bootable USB Drive
-------------
To install arch, we will first create a bootable USB drive. The [official documentation] for this process is very well developed, but we will provide an example using linux or macOS

###From a Mac

1. Download the arch linux .iso from https://www.archlinux.org/download/. Keep track of the download path using a bash variable.
   
   ```bash
   # NOTE: Replace the path below with the appropriate one for you
   download_path=~/Downloads/archlinux-2018.02.01-x86_64.iso
   ```

2. Plug in your USB drive. It should go without saying that you must be willing to lose **all** data saved on the drive.

3. Determine which device points to your USB drive (look for the one whose size matches your drive).

   ```bash
   diskutil list
   # NOTE: change the below disk number to that identified with the above command
   usb_device_number=2
   ```

3. Unmount your USB device.

   ```bash
   diskutil unmountDisk /dev/disk$usb_device_number
   ```

4. Write the .iso image to your device using dd. Note that we are using /dev/rdiskx not /dev/diskx. This is because it is much faster on OS X. 1M Refers to the block size to be written.

  ```bash
   dd if="$download_path" of=/dev/rdisk$usb_device_number bs=1M
   ```

5. Eject your USB drive, you're done! You should always safely eject media before removing it.

   ```bash
   diskutil eject /dev/disk$usb_device_number
   ```
   
   ###From Linux
   
1. Download the arch linux .iso from https://www.archlinux.org/download/. Keep track of the download path using a bash variable.
   
   ```bash
   # NOTE: Replace the path below with the appropriate one for you
   download_path=~/Downloads/archlinux-2018.02.01-x86_64.iso
   ```

2. Plug in your USB drive. It should go without saying that you must be willing to lose **all** data saved on the drive.

3. Determine which device (e.g., "/dev/sdc") points to your USB drive (look for the one whose size matches your drive).

   ```bash
   # You can optionally try lsblk -f or lsblk -S to get more info about device filesystem type or hardware model
   lsblk
   # NOTE: change the below device path that identified with the above command
   # Note: this should *not* include the number of any mounted partition (e.g., /dev/sdc1)
   usb_device_path=/dev/sda
   ```

3. Write the .iso image to your device using dd. 4m Refers to the block size to be written.

  ```bash
   dd if="$download_path" of=$usb_device_path bs=4M
   ```

4. Eject your USB drive, you're done! You should always safely eject media before removing it.

   ```bash
   eject $usb_device_path
   ```

Installing Arch Linux
-------------
### Booting into Your Arch Live USB

1. (Optional) Disable all irrelevant HDDs/SSDs using your motherboard's settings. You will need to find specific instructions for your motherboard. Generally, this will simply involve toggling a SATA port' enabled state. When I am installing a system on a drive, I like it to be the only one that can possibly be touched. This prevents accidentally choosing the wrong drive while partitioning, and stops frustrating bootloader overwrites that can occur if windows is on your lowest-numbered SATA port.

2. Plug your USB drive into your computer, turn it on, and enter your motherboard's alternative boot menu (usually achieved by spamming F12 as soon as you have finished pressing the power button). Select the device with your USB drive's manufacturer followed by "UEFI." After a brief loading screen, you should be booted into your live USB as the root user.

### Installing Arch from your Live USB
#### Prerequisites and Helpful Tools

1. Confirm EFI support is working

  ```bash
   ls /sys/firmware/efi/efivars
   ```

  If the above fails to show a directory filled with items, either your motherboard does not support UEFI, or you did not correctly select the UEFI boot option at your motherboard's alternative boot menu. **Do not proceed with this guide until this step is successful.**

2. Update Arch pacman repoitories

  ```bash
   pacman -Syy
   ```

3. (Optional) Install and use reflector to optimize the Arch mirrors your system connects to. This can *greatly* increase your download speeds throughout this guide. Update the country information to fit your situation.

   ```bash
   reflector --verbose --country 'United States' -l 200 --sort rate --save /etc/pacman.d/mirrorlist
   ```

4. (Optional) Install an ssh server onto the live image so that you can run commands on your system from another computer (really helps to have copy paste support / view a terminal next to a web browser for support).

  ```bash
   # Install openssh while updating source repositories
   pacman -Sy openssh
   # Set a password for the USB's root user (required for logging in via ssh)
   # Note: this password will not apply to the installed system, only the temporary live USB
   passwd
   # <enter your password at the prompt>
   # Tell arch to start the openssh server
   systemctl start sshd
   # Determine the IP address of your system (it will likely look like 192.168.1.XX )
   ip addr | grep inet
   ```

   ssh into your system, e.g., `ssh root@192.168.1.30` and continue the following steps from that terminal.
   
   #### Setting up Partitions
   
5. Determine which drive you intend to install Arch on (look for the one who's size matches your drive). I generally disable all other drives.

  ```bash
  # Show available devices with file system types, labels, UUIDs, and mountpoints 
   lsblk -f
   # Show available devices with model numbers
   lsblk -S
   # NOTE: replace the below with the correct value for you
   # ----IF YOU USE THE WRONG VALUE, YOU WILL BREAK YOUR SYSTEM----
   target_device=/dev/sda
   ```

6. Use parted to create a partition table, a UEFI boot partition, and create a BTRFS  partition for the rest of the system. We will be using parted in interactive mode for this guide, but many other tools exist. We are not creating a swap partition, but if you think you may run low on RAM, you should.

   ```bash  
   # Enter parted interactive mode
   parted $target_device`
   ```

   The below commands will be in parted interactive mode where terminal commands are prefaced by "(parted)"
   
   ```bash    
   # Tells parted to use the gpt partioning table
   mklabel gpt
   # Set the first partition to be the ESP/EFI partition required by UEFI bootloaders
   # The EFI partition *must* be fat32 and at least 512MiB
   mkpart ESP fat32 1MiB 513MiB
   # Enable the boot flag for the ESP boot partition
   set 1 boot on
   # Fill the rest of the drive with a btrfs partition
   mkpart primary btrfs 513MiB 100%
   # Save changes and exit
   quit
   ```

7. Create the filesystems for the new partitions. We will make the first partition of our device the boot partition and the second one the BTRFS partition.

   ```bash
   # Format paritions by creating file systems
   # Create UEFI boot fat32 file system
   # $(echo $target_device)1 simply turns "/dev/sdc" into "/dev/sdc1" for example
   sudo mkfs.fat -F32 $(echo $target_device)1
   # Create root btrfs file system
   sudo mkfs.btrfs -L "Arch Linux" $(echo $target_device)2
   ```

8. Temporarily mount the newly formatted BTRFS partition (this is the toplevel of the drive, we want to install into a subvolume child of this toplevel).
   
   ```bash
   # Create the directory for the mount point
   mkdir -p /mnt/new_system_root
   # Mount your btrfs partition to /mnt
   # e.g.: mount /dev/sda2 /mnt/new_system_root
   mount $(echo $target_device)2 /mnt/new_system_root
   ```

9. Create BTRFS subvolumes for easy snapshotting and restoring with snapper
[](TODO: confirm btrfs-progs ships wtih live cd)

   ```bash
   cd /mnt/new_system_root/
   # Root ("/") subvolume
   btrfs subvolume create /subvol_root
   # Home ("/home") subvolume
   btrfs subvolume create /subvol_home
   # Root snapshots/backup subvolume
   btrfs subvolume create /subvol_root_snapshots
   # Home snapshots/backup subvolume
   btrfs subvolume create /subvol_home_snapshots
   cd /mnt
   # Unmount the BTRFS partition so we can mount only the root partition for our Arch install
   umount /mnt/new_system_root
   ```

10. Mount the root-only BTRFS subvolume to hold our Arch install.

   ```bash
   # e.g.: mount -o subvol=/subvol_root sda2 /mnt/new_system_root
   mount -o subvol=/subvol_root $(echo $target_device)2 /mnt/new_system_root
   mkdir /mnt/new_system_root/boot
   # Mount your fat32 UEFI ESP partition
   # e.g.: mount /dev/sda1 /mnt/new_system_root/boot
   mount $(echo $target_device)1 /mnt/new_system_root/boot
   ```
   
11. Install base pacakges into root-only BTRFS subvolume

   ```bash
   pacstrap -i /mnt/new_system_root/ base base-devel
   ```
   
12. Create an initial fstab file.

   ```bash
   #Create initial fstab file.
   genfstab -U /mnt/new_system_root/ >> /mnt/new_system_root/etc/fstab
   ```

13. Modify the initial fstab file to mount our subvolumes as snapper expects.

   First,
   
   ```bash
   nano /mnt/new_system_root/etc/fstab
   ```
   
   Then, modify the file to be **similar,** NOT identiical, to the following (ctrl+o saves and ctrl+x exits):
   
   ```bash
   # /etc/fstab: static file system information
   #
   # <file system> <dir>   <type>  <options>       <dump>  <pass>
   
   # UEFI ESP Boot Partition
   # /dev/sda1
   UUID=765A-EB3A          /boot           vfat            rw,nosuid,nodev,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,utf8,errors=remount-ro       0 2
   
   # BTRFS Root Partition /dev/sda2 LABEL=Arch Linux
   # Root subvolume
   UUID=d84a9588-276b-4df5-8282-4f1426a87090       /               btrfs           rw,nodev,relatime,ssd,discard,space_cache,subvol=/subvol_root 0 0
   
   # Home subvolume
   UUID=d84a9588-276b-4df5-8282-4f1426a87090       /home           btrfs           rw,nosuid,nodev,relatime,ssd,discard,space_cache,subvol=/subvol_home  0 0
   
   # Snapshot subvolumes mounted where expected by snapper
   # Root snapshots
   UUID=d84a9588-276b-4df5-8282-4f1426a87090       /.snapshots     btrfs          rw,nosuid,nodev,relatime,ssd,discard,space_cache,subvol=subvol_root_snapshots       0 0
   # Home snapshots
   UUID=d84a9588-276b-4df5-8282-4f1426a87090       /home/.snapshots        btrfs  rw,nosuid,nodev,relatime,ssd,discard,space_cache,subvol=subvol_home_snapshots       0 0
   ```
   The relevant information is that for each subvolume, there should be a line mounting that subvolume (specificed by "subvol=") with the correct UUID of your SSD (or HDD) and the correct mountpoint. The flags included in this example should be acceptable for all SSD configurations. If you are using an HDD, do not use the SSD flag--it is likely fine to keep the default flags from the initial fstab for all lines.
   
   #### Configuring the Initial Install
   
14. Change root into your system for configuration. Up until this point, the root perspective has been that of the live USB drive. Now, we will change root into our newly created system. The path referenced as /mnt/new_system_root/ before this command will be accessible as / after this command.

   ```bash
   arch-chroot /mnt/new_system_root/ /bin/bash
   ```

15. Set the timezone
   ```bash
   # NOTE: replace below timezone with that appropriate to you
   ln -sf /usr/share/zoneinfo/America/New_york /etc/localtime
   hwclock --systohc
   ```

16. Set the locale and language
   ```bash
   # NOTE: replace the below value with that appropriate to you
   desired_locale="en_US.UTF-8 UTF-8"
   echo "$desired_locale" >>/etc/locale.gen
   # Note: the above commands can be replaced by using "nano /etc/locale.gen" and uncommenting the line corresponding to your locale
   # Generate locales
   locale-gen
   # Set Lang variable
   # NOTE: replace the below value with that appropriate to you
   desired_lang="en_US.UTF-8"
   echo "LANG=$desired_lang" > /etc/locale.conf
   ```

17. Set the hostname
   ```bash
   # NOTE: replace the below value with that appropriate to you
hostname="Arch"
echo "$hostname" > /etc/hostname
   ```

18. Set up the hosts file
   ```bash
   echo -e "127.0.0.1\tlocalhost" >> /etc/hosts
   echo -e "::1\t\tlocalhost" >> /etc/hosts
   echo -e "127.0.1.1\t$hostname.localdomain\t$hostname" >> /etc/hosts
   ```

19. Determine your ethernet interface's device name.

   ```bash
   ls /sys/class/net
   ```
 A device following a naming scheme starting with the letters "en" should be your ethernet adapter (e.g., eno1). This device name is what will be used in the next step. (We are not interested in the device named "lo").

20. Enable wired networking. Although the internet is working via the live USB, booting into the system will not have internet access without explicitly enabling it. Replace eno1 with the relevant the device name from the previous step
   ```bash
   # NOTE: replace "eno1"" with the relevant device name from the previous step
   systemctl enable dhcpcd@eno1.service
   ```

21. Create a pre-init ramdisk environment.

   ```bash
   mkinitcpio -p linux
   ```

22. Download any microcode updates for your hardware. Occasionally, hardware makers make a few mistakes. Microcode updates are patches to fix these. This guide details patching an Intel-based processor. It is possible there are no microcode updates for your processor (Intel or otherwise). Note: this must be done before configuring the bootloader to take effect.

  ```bash
   pacman -S intel-ucode
   ```
   
 #### Setting up the Grub Bootloader

23. Configure the Grub bootloader.
   ```bash
   # Install required packages
   pacman -S grub efibootmgr
   # Install grub to our EFI boot partition
   grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub
   # Modify our grub config to boot the correct BTRFS subvolume
   # Adds "rootflags=subvol=subvol_root" to GRUB_CMDLINE_LINUX_DEFAULT  in /etc/default/grub
   sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="/GRUB_CMDLINE_LINUX_DEFAULT="rootflags=subvol-subvol_root /g' /etc/default/grub
   # Generate our grub.cfg
   grub-mkconfig -o /boot/grub/grub.cfg
   ```
24. Set the password for the system's root account (the previously set root password was **only** for the live USB).
   ```bash
   passwd
   # <enter your desired password> Note: typed characters will not display on screen, this is normal for linux)
   ```
25. Reboot and confirm you are able to login as the root user (note, the root user will not be acessible via ssh due to openssh default settings--create a new user before rebooting if you wish to continue remotely)

   ```bash
   reboot
   ```

Configuring an Arch Linux Install
-------------
1. Install zsh

 By default, most systems come with bash as the default shell. Do you remember the cool shell from the live USB that allowed for case-insensitive tab completions and a whole array of other cool features? We're going to set that up. We'll also install git while we're at it.

  ```bash
   pacman -Sy zsh git
   ```
### Configuring an Admin User Account
   
2. Create an admin user

 Your Arch installation currently only has a root account, this is almost universally a bad idea, let's set up another user. We will use the user name "admin," but you should replace this with your own. We are adding this user to the wheel group because we are going to set it up as a system admin. The default shell will be zsh, which we just installed.

  ```bash
  # Add a new user named "admin" to the wheel group with zsh as the default shell
   useradd -m -G wheel -s /bin/zsh admin
   #<type in a password for admin>
   # Grant sudo access to the sudoers group
   # The sed command is simply uncommenting the line %wheel ALL=(ALL) ALL
   # Note: if you intend to add multiple admin users, thhe below command should only be run once
   sed -i 's/#%wheel ALL=(ALL) ALL/%wheel ALL=(ALL) ALL/g' /etc/sudoers
   ```

3. Login to our new user. **From this point on, we don't want to be logged in as root if we don't need to be. Instead, we will use our new admin account, "admin," and simply use `sudo` when necessary.**

  ```bash
   su admin
   # <type in desired admin password>
   ```

4. Setup [oh my zsh](https://github.com/robbyrussell/oh-my-zsh).

   ```bash
   sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
   ```

5. Setup [Trizen](https://github.com/trizen/trizen). AUR helpers can be immensely convenient for installing user contributed packages. Keep in mind AUR packages are not official and can have their bugs/issues.
   ```bash
   install trizen
   git clone https://aur.archlinux.org/trizen.git
   cd trizen
   makepkg -si
   ```

### Configuring Snapper and BTRFS snapshots

1. Install [snapper](https://wiki.archlinux.org/index.php/Snapper) and [snap-pac](https://github.com/wesbarnett/snap-pac). Snap-pac takes automatic snapshots every time pacman (or an AUR helper calling it) installs/uninstalls a package.

   ```bash
   sudo pacman -S snapper snap-pac
   ```

2. Set up initial snapper snapshots config for the root subvolume
   ```bash
   # Temporarily unmount the /.snapshots subvolume
   sudo umount /.snapshots
   sudo rmdir /.snapshots
   # Create an initial config file for the root subvolme
   sudo snapper -c subvol_root_config create-config /
   # Delete the subvolume created by snapper
   sudo btrfs subvolume delete /.snapshots
   # Restore the subvol_root_snapshots subvolume
   sudo mkdir /.snapshots
   # Only the mount path is needed because the path is in the fstab file
   sudo mount /.snapshots
   ```
3. Set up initial snapper snapshots config for the home subvolume
   ```bash
   # Temporarily unmount the /home/.snapshots subvolume
   sudo umount /home/.snapshots
   sudo rmdir /home/.snapshots
   # Create an initial config file for the root subvolme
   sudo snapper -c subvol_home_config create-config /home
   # Delete the subvolume created by snapper
   sudo btrfs subvolume delete /home/.snapshots
   # Restore the subvol_root_snapshots subvolume
   sudo mkdir /home/.snapshots
   # Only the mount path is needed because the path is in the fstab file
   sudo mount /home/.snapshots
   ```
4. Configure root snapshots config
   
   ```bash
   # Change max space limit from 50% of disk space to 15%
   sudo sed -i 's/SPACE_LIMIT="0.5"/SPACE_LIMIT="0.15"/g' /etc/snapper/configs/subvol_root_config
   # Change max count of "number"" snapshots to 20
   sudo sed -i 's/NUMBER_LIMIT="50"/NUMBER_LIMIT="20"/g' /etc/snapper/configs/subvol_root_config
   # Keep 12 hourly snapshots
   sudo sed -i 's/TIMELINE_LIMIT_HOURLY="10"/TIMELINE_LIMIT_HOURLY="12"/g' /etc/snapper/configs/subvol_root_config
   # Keep 7 daily snapshots
   sudo sed -i 's/TIMELINE_LIMIT_DAILY="10"/TIMELINE_LIMIT_DAILY="7"/g' /etc/snapper/configs/subvol_root_config 
   # Keep 1 monthly snapshot
   sudo sed -i 's/TIMELINE_LIMIT_MONTHLY="10"/TIMELINE_LIMIT_MONTHLY="1"/g' /etc/snapper/configs/subvol_root_config 
   # Keep no annual snapshots
   sudo sed -i 's/TIMELINE_LIMIT_YEARLY="10"/TIMELINE_LIMIT_YEARLY="0"/g' /etc/snapper/configs/subvol_root_config
   # Add automatic snapshotting with snap-pac
   sudo cp /etc/snap-pac/ root.conf.example subvol_root_config.conf
   echo 'SNAPSHOT="yes"' | sudo tee -a /etc/snapper/configs/subvol_root_config > /dev/null 
   ```
5. Configure home snapshots config

   ```bash
   # Change max space limit from 50% of disk space to 15%
   sudo sed -i 's/SPACE_LIMIT="0.5"/SPACE_LIMIT="0.15"/g' /etc/snapper/configs/subvol_home_config
   # Change max count of "number"" snapshots to 20
   sudo sed -i 's/NUMBER_LIMIT="50"/NUMBER_LIMIT="20"/g' /etc/snapper/configs/subvol_home_config
   # Keep 12 hourly snapshots
   sudo sed -i 's/TIMELINE_LIMIT_HOURLY="10"/TIMELINE_LIMIT_HOURLY="12"/g' /etc/snapper/configs/subvol_home_config
   # Keep 7 daily snapshots
   sudo sed -i 's/TIMELINE_LIMIT_DAILY="10"/TIMELINE_LIMIT_DAILY="7"/g' /etc/snapper/configs/subvol_home_config 
   # Keep 1 monthly snapshot
   sudo sed -i 's/TIMELINE_LIMIT_MONTHLY="10"/TIMELINE_LIMIT_MONTHLY="1"/g' /etc/snapper/configs/subvol_home_config 
   # Keep no annual snapshots
   sudo sed -i 's/TIMELINE_LIMIT_YEARLY="10"/TIMELINE_LIMIT_YEARLY="0"/g' /etc/snapper/configs/subvol_home_config
   # Add automatic snapshotting with snap-pac
   sudo cp /etc/snap-pac/ root.conf.example subvol_home_config.conf
   ```

6. 
   
   
configure nano
set nowrap ~/.nanorc

nvidia, zcash, ncdu, 

likes:
nethogs, htop,