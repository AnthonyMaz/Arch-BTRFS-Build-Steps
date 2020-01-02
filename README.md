# Arch-BTRFS-Build-Steps
============

####Partitioning drive
```bash
gdisk /dev/sdX
```
Select new GPT table (option o)
* Boot partition:
  * option n, +800M, label as ef02
* Second partition (for btrfs subvolumes)	 
  * option n, Desired size, label 8300 (default) .Leave room for swap if needed)
  * Can just press enter when picking size to use the rest of the disk
* Third partition (swap if needed)
  * option n, Desired size, label as 8200
* Press w to write changes 

To see partitions:
```bash
lsblk
```

To make partitions
```base
mkfs.fat -F32 /dev/sda1
mkfs.btrfs /dev/sda2
mkswap /dev/sda3
```

Mount the root btrfs volume
```bash
mount /dev/sda2 /mnt
```
Create subvolume for root, home, var and one for snapshots
```bash
btrfs subvolume create /mnt/@root
btrfs subvolume create /mnt/@var
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots
```

Mount them
```
umount /mnt
mount -o noatime,compress=lzo,space_cache,subvol=@root /dev/sda2 /mnt
mkdir /mnt/{boot,var,home,.snapshots}
mount -o noatime,compress=lzo,space_cache,subvol=@var /dev/sda2 /mnt/var
mount -o noatime,compress=lzo,space_cache,subvol=@home /dev/sda2 /mnt/home
mount -o noatime,compress=lzo,space_cache,subvol=@snapshots /dev/sda2 /mnt/.snapshots
```

Pick closest mirror
```bash
vim /etc/pacman.d/mirrorlist
```
Delete every except for the mirrors in your country (press 'dd' to delete the current line, or `1000dd` to delete the next 1,000 lines)


Install base system, bootloader, and other useful packages
```bash
pacstrap /mnt base base-devel grub vim networkmanager wget htop lsof dkms openssh mkinitcpio linux linux-firmware
```


Install other packages
```bash
pacstrap /mnt 
```

Generate fstab
```bash
genfstab -p /mnt >> /mnt/etc/fstab 
```

Chroot into new system
```bash
arch-chroot /mnt 
```

Create user in the group wheel 
```bash
useradd -m -G wheel username
```

Set user password
```bash
passwd username
```

Set root password
```bash
passwd 
```

Write hostmane to /etc/hostname
```bash
echo "MyComputerName" > /etc/hostname
```

Set up timezone
```bash
ln -s /usr/share/zoneinfo/US/Eastern /etc/localtime
```

Edit /etc/locale.gen and enable English UTF-8 with US Settings by uncommenting the line: 
```
en_US.UTF-8 UTF-8
```

Generate locale.
```bash
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

Set local time
```bash
tzselect
```

Create an initial ramdisk.
```bash
mkinitcpio -p linux
```
If the below errors are seen, they can be ignored. 
		==> ERROR: file not found: `fsck.btrfs'
		==> WARNING: No fsck helpers found. fsck will not be run on boot.

Setup grub.
```bash
grub-install --target=i386-pc /dev/sdX
```

Generate /boot/grub/grub.cfg	
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

Install Gnome, enabling gdm and NetworkManager (select the default options)
```bash
pacman -S xorg-server xorg-xinit xorg-apps gnome 
systemctl enable gdm.service
systemctl enable NetworkManager.service
```

A few other packages to install.
```bash
pacman -S ttf-dejavu xorg-twm xterm alsa-utils gnome-tweaks zsh
```

After installing any other packages, exit arch-chroot, unmount all subvolues and partitions, then restart to your Arch install.
```bash
exit
cd /
umount /mnt/var
umount /mnt/home
umount /mnt/.snapshots
umount /mnt
```

To double check for any other paritions mounted:
```bash
lsblk -f
```

Reboot (You must power off, remove the arch ISO install media, and restart)
```bash
shutdown -h now
```

When you log into your arch install, configure wheel and vim.
```bash
visudo
```

Uncomment this line to allow members of group wheel to execute any command:
```
%wheel ALL=(ALL) ALL
```
	
Same thing without a password
```
%wheel ALL=(ALL) NOPASSWD: ALL
:wq
```

Set the default text editor:
```
vim /etc/environment		
EDITOR=/usr/bin/vim
:wq
```

A few things you can add to your .bashrc
```bash
alias ls='ls --color=auto'
PS1='[\u@\h \W]\$ '
alias ll='ls -l'
alias grep='grep --color=always'
export HISTTIMEFORMAT="%h %d %H:%M:%S "
alias snap-home='sudo btrfs subvolume snapshot /home/ /.snapshots/$(date +%m-%d-%Yhome)'
alias snap-var='sudo btrfs subvolume snapshot /var/ /.snapshots/$(date +%m-%d-%Yvar)'
alias snap-root='sudo btrfs subvolume snapshot / /.snapshots/$(date +%m-%d-%Yroot)'
```

**Note: Double check your fstab. For some reason on a previous install, it didn't generate correctly.
Just boot off the iso and mount everything, and run genfstab.**
```bash
mount -o noatime,compress=lzo,space_cache,subvol=@root /dev/sda2 /mnt
mount -o noatime,compress=lzo,space_cache,subvol=@var /dev/sda2 /mnt/var
mount -o noatime,compress=lzo,space_cache,subvol=@home /dev/sda2 /mnt/home
mount -o noatime,compress=lzo,space_cache,subvol=@snapshots /dev/sda2 /mnt/.snapshots
genfstab -p /mnt >> /mnt/etc/fstab
```

You can create snapshots with the below command.I recommend just using an alias for each subvolume like above.
```bash
btrfs subvolume snapshot -r / "/.snapshots/@root-$(date +%F-%R)"
```

To restore from snapshot you just delete the currently used subvolume and replace it with an earlier snapshot.

After loading from the archiso USB, run the below.
```bash
mount /dev/sda2 /mnt
btrfs subvolume delete /mnt/@root
brtfs subvolume snapshot /mnt/@snapshots/@root-2019-11-10-20:19 /mnt/@root
```
