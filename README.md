# Arch-BTRFS-Build-Steps
============

Partitioning drive
```
	gdisk /dev/sdX 
	select new GPT table (option o)
	Boot partition
		option n, +800M, label as ef02
	Second partition (for btrfs subvolumes)	 
		option n, Desired size, label 8300 (default) .Leave room for swap if needed)
	Third partition (swap if needed)
		option n, Desired size, label as 8200
	then press w to write changes 
```
Format partitions
```
	lsblk to see partitions
	mkfs.fat -F32 /dev/sda1
	mkfs.btrfs /dev/sda2
	mkswap /dev/sda3
```
BTRFS subvolumes:

Mount the root btrfs volume
```
	mount /dev/sda2 /mnt
```
Create subvolume for root, home, var and one for snapshots.
```
	btrfs subvolume create /mnt/@root
	btrfs subvolume create /mnt/@var
	btrfs subvolume create /mnt/@home
	btrfs subvolume create /mnt/@snapshots
```
Mount them.
```
	umount /mnt
	mount -o noatime,compress=lzo,space_cache,subvol=@root /dev/sda2 /mnt
	mkdir /mnt/{boot,var,home,.snapshots}
	mount -o noatime,compress=lzo,space_cache,subvol=@var /dev/sda2 /mnt/var
	mount -o noatime,compress=lzo,space_cache,subvol=@home /dev/sda2 /mnt/home
	mount -o noatime,compress=lzo,space_cache,subvol=@snapshots /dev/sda2 /mnt/.snapshots
```
Install base system

```
	pacstrap /mnt base base-devel
```
Install bootloader
```
	pacstrap /mnt grub
```
Install other packages
```
	pacstrap /mnt vim networkmanager curl wget htop lsof sudo dkms openssh mkinitcpio linux linux-firmware
```
Generate fstab
```
	genfstab -p /mnt >> /mnt/etc/fstab 
```
Chroot into new system
```	
	arch-chroot /mnt 
```
Create user in the group wheel 
```
	useradd -m -G wheel username
```
Set user password.
```
	passwd username
```
Set root password.
```
	passwd 
```
Write hostmane to /etc/hostname.
```
	echo "MyComputerName" > /etc/hostname
```
Set up locale settings.
```
	ln -s /usr/share/zoneinfo/US/Eastern /etc/localtime
```
Edit /etc/locale.gen and uncomment line 176 
	Enable UTF-8 with US Settings. 
	en_US.UTF-8 UTF-8
Generate locale.
```
	locale-gen
	echo "LANG=en_US.UTF-8" > /etc/locale.conf
```
Set local time.
```
	tzselect, and set to your time zone
```
Create an initial ramdisk.
```
	mkinitcpio -p linux
	If the below errors are seen, they can be ignored. 
		==> ERROR: file not found: `fsck.btrfs'
		==> WARNING: No fsck helpers found. fsck will not be run on boot.
```
Setup grub.
```
	grub-install --target=i386-pc /dev/sdX
```
Generate /boot/grub/grub.cfg	
```
	grub-mkconfig -o /boot/grub/grub.cfg
```
Install a Gnome, enabling gdm and NetworkManager
```
	pacman -S xorg-server xorg-xinit xorg-apps (select default)
	pacman -S gnome (select default)
	systemctl enable gdm.service
	systemctl enable NetworkManager.service
```
A few other packages to install.
```
	pacman -S ttf-dejavu xorg-twm xterm alsa-utils pulseaudio gnome-tweaks zsh which
```
After installing any other packages, exit arch-chroot, unmount all subvolues and partitions, then restart to your Arch install.
```
	exit
	cd /
	umount /mnt/var
	umount /mnt/home
	umount /mnt/.snapshots
	umount /mnt
	lsblk -f (to double check for any other paritions mounted)
```
When you log into your arch install, configure wheel and vim.
```
	# visudo
	## Uncomment to allow members of group wheel to execute any command
	%wheel ALL=(ALL) ALL
	
	## Same thing without a password
	%wheel ALL=(ALL) NOPASSWD: ALL
	:wq
```
```
	# vim /etc/environment		
	EDITOR=/usr/bin/vim
	:wq
```

A few things you can add to your .bashrc and/or .zshrc
```
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
```
	mount -o noatime,compress=lzo,space_cache,subvol=@root /dev/sda2 /mnt
	mount -o noatime,compress=lzo,space_cache,subvol=@var /dev/sda2 /mnt/var
	mount -o noatime,compress=lzo,space_cache,subvol=@home /dev/sda2 /mnt/home
	mount -o noatime,compress=lzo,space_cache,subvol=@snapshots /dev/sda2 /mnt/.snapshots
	genfstab -p /mnt >> /mnt/etc/fstab
```

You can create snapshots with the below command.I recommend just using an alias for each subvolume like above.
```
	btrfs subvolume snapshot -r / "/.snapshots/@root-$(date +%F-%R)"
```
To restore from snapshot you just delete the currently used subvolume and replace it with an earlier snapshot.

After loading from the archiso USB, run the below.
```
	mount /dev/sda2 /mnt
	btrfs subvolume delete /mnt/@root
	brtfs subvolume snapshot /mnt/@snapshots/@root-2019-11-10-20:19 /mnt/@root
```
