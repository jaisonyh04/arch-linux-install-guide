<img width="1018" height="766" alt="Plasma-Desktop" src="https://github.com/user-attachments/assets/17b39f91-872c-4b96-9c09-9a3bc088326e" />

# Guide description 

A step-by-step Arch Linux installation guide for a **UEFI system using GPT, ext4, Limine, and the linux-zen kernel**. It covers disk partitioning, base system configuration, networking, ZRAM, users and sudo, automated Limine kernel management with `limine-tool`, and a functional KDE Plasma desktop. It also includes `linux-lts` as a fallback kernel and essential system services and utilities.


>**UEFI requirement:** This guide is intended exclusively for systems booted in UEFI mode with secure boot disabled. 


# Boot the ISO

Verify the boot mode before continuing:


```
cat /sys/firmware/efi/fw_platform_size
```


Checking Boot Mode:

- **UEFI Mode (64-bit):** If the command returns `64`, your system is booted in UEFI mode with a 64-bit x64 UEFI.

- **UEFI Mode (32-bit):** If the command returns `32`, your system is booted in UEFI mode with a 32-bit IA32 UEFI.

- **BIOS/CSM Mode:** If the command returns an error like `no such file or directory`, your system is booted in legacy BIOS mode. 


Alternatively, you can check for the existence of EFI variables by listing the directory contents with:


```
ls /sys/firmware/efi/efivars
```



>If the directory exists and displays its contents, the system was booted in UEFI mode. If it does not exist, reboot and start the Arch installation medium using its UEFI entry.


# Temporarily change the font for the Linux virtual console:


```
setfont ter-v24b
```


# Verify internet


```
ping -c 3 archlinux.org
```


>If it works, you're ready.


# Identify the disk where Arch Linux will be installed


```
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL
```

For example:

```
/dev/nvme0n1
/dev/sda
/dev/vda
```


>Verify the disk model and capacity carefully. Selecting the wrong disk during partitioning or formatting can permanently destroy data.


# Partition the disk


Replace `<disk>` with your target disk:


```
cfdisk /dev/<disk>
```


Example:


```
cfdisk /dev/nvme0n1
```


Create a **GPT** partition table with:

| Partition            | Size            | Type             |
| -------------------- | --------------- | ---------------- |
| EFI System Partition | 6 GiB           | EFI System       |
| Root (`/`)           | Remaining space | Linux filesystem |

<img width="1021" height="767" alt="cfdisk" src="https://github.com/user-attachments/assets/9ef1eea5-a0ff-4d1d-9e7f-2a217124b200" />


# Format the partitions


Replace the placeholders with the partitions created above:


```
mkfs.fat -F32 /dev/<efi-partition>
mkfs.ext4 /dev/<root-partition>
```


Example for an NVMe drive:


```
mkfs.fat -F32 /dev/nvme0n1p1
mkfs.ext4 /dev/nvme0n1p2
```


# Mount the filesystems


```
mount /dev/<root-partition> /mnt

mkdir /mnt/boot

mount /dev/<efi-partition> /mnt/boot
```


# Install the base system


```
pacstrap -K /mnt \
  base base-devel git \
  linux-zen linux-zen-headers \
  linux-lts linux-lts-headers \
  linux-firmware zram-generator vim nano sudo \
  limine efibootmgr
```


>**CPU microcode**: During `pacstrap`, add `intel-ucode` for an Intel processor or `amd-ucode` for an AMD processor. Install only the package that matches your CPU.


# Generate fstab


```
genfstab -U /mnt >> /mnt/etc/fstab
```


Check:


```
cat /mnt/etc/fstab
```


# Chroot


```
arch-chroot /mnt
```


>At this point, you've successfully installed the base system onto the disk. The next steps are configuring the installed system so it can boot on its own.

# zram-generator


```
nano /etc/systemd/zram-generator.conf
```



```
[zram0] 
zram-size = ram / 2 
compression-algorithm = zstd 
swap-priority = 100
```

Save and exit (Ctrl + O, Enter, Ctrl + X).


# Set time zone

Find your time zone::


```
timedatectl list-timezones
```


For example:


```
America/New_York 
Europe/London 
Asia/Tokyo
```


Set the time zone by replacing `<Region>/<City>` with your correct time zone:


```
ln -sf /usr/share/zoneinfo/<Region>/<City> /etc/localtime
```


Generate `/etc/adjtime`:


```
hwclock --systohc
```


Verify:


```
date
```


# Configure the locale

Open the file:


```
nano /etc/locale.gen
```


Find the locales you want and uncomment them by removing the #.

For an English system, uncomment:


```
en_US.UTF-8 UTF-8
```


Save and exit (Ctrl + O, Enter, Ctrl + X).

## Generate the locales


```
locale-gen
```


You should see output similar to:


```
Generating locales...

  en_US.UTF-8... done

  Generation complete.
```


## Set the default locale

If you want your system in English, run:


```
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```


You can check the file with:


```
cat /etc/locale.conf
```


Expected output:


```
LANG=en_US.UTF-8
```


# Set the hostname

Replace `<hostname>` with the desired hostname:


```
echo "<hostname>" > /etc/hostname
```


Example:


```
echo "archlinux" > /etc/hostname
```


Verify:


```
cat /etc/hostname
```


# Configure /etc/hosts


```
nano /etc/hosts
```


Add the following:


```
127.0.0.1   localhost

::1         localhost

127.0.1.1   <hostname>.localdomain <hostname>

```


Save and exit (Ctrl + O, Enter, Ctrl + X).

Display the file:


```
cat /etc/hosts
```


Expected output:


```
127.0.0.1   localhost

::1         localhost

127.0.1.1   archlinux.localdomain archlinux
```


# Set the root password


```
passwd
```


>You'll be prompted to enter and confirm a password. As you type, **nothing will appear on the screen**, this is normal in Linux.

# Create the user

Replace `<username>` with the desired username:


```
useradd -m -G wheel -s /bin/bash <username>
```


What the options mean:

-m = Create a home directory (/home/username)

-G wheel = Add the user to the wheel group (needed for sudo)
  
-s /bin/bash = Set Bash as the default shell

# Set your user's password


```
passwd <username>
```


Again, enter and confirm the password.

# Allow the wheel group to use sudo

Open the sudoers file **using** **visudo** (never edit it directly with nano):


```
EDITOR=nano visudo
```


Find this line:


```
#%wheel ALL=(ALL:ALL) ALL
```


Remove the # so it becomes:


```
%wheel ALL=(ALL:ALL) ALL
```


Save and exit (Ctrl + O, Enter, Ctrl + X).

>visudo checks the file for syntax errors before saving, which helps prevent accidentally breaking sudo.

Test (optional)

You can confirm your user exists:


```
id <username>
```


>You should see that **username** belongs to the wheel group.


# Install limine-tool

Switch to your newly created user account:


```
su - <username>
```



```
sudo pacman -S --needed git base-devel rust
```



```
git clone https://github.com/CatOS-dev/limine-tool.git
cd limine-tool/packaging/arch
makepkg -si
```



Verify


```
pacman -Q limine-tool
```



```
command -v limine-install
command -v limine-mkinitcpio
command -v limine-entry-tool
```



>You should get paths under `/usr/bin/`. The package provides these commands through a single `/usr/lib/limine/limine-tool` multicall executable.


Return to root:


```
exit
```


# Verify ESP detection:


```
bootctl --print-esp-path
```


Expected:


```
/boot
```


# Configure Limine

First obtain the **PARTUUID** of the **root partition**:


```
blkid -s PARTUUID -o value /dev/<root-partition>
```


```
lsblk -o NAME,FSTYPE,PARTUUID,MOUNTPOINTS
```


Then create:


```
nano /etc/default/limine
```


with:


```bash
ESP_PATH=/boot
KERNEL_CMDLINE[default]=root=PARTUUID=YOUR_PARTUUID rw quiet
KERNEL_CMDLINE[linux-lts]=root=PARTUUID=YOUR_PARTUUID rw
FIND_BOOTLOADERS=yes
```


Save and exit (Ctrl + O, Enter, Ctrl + X).


>Replace YOUR_PARTUUID with the PARTUUID from blkid.


# Install Limine into the EFI System Partition


```
limine-install
```


>`limine-install` reads its configuration from `/etc/default/limine`. In this guide, `ESP_PATH=/boot` is configured beforehand so the correct EFI System Partition is used. If `ESP_PATH` is omitted, `limine-install` will attempt to detect the ESP automatically.


Generate the initial configuration and kernel entries:


```
limine-update
```


Verify:

```
limine-list 
cat /boot/limine.conf 
efibootmgr --verbose
```


>The tool automatically creates entries for both `linux-zen` and `linux-lts`using your current `mkinitcpio` presets. Future kernel updates regenerate the entries automatically.


# General Desktop and Utilities


## Functional Plasma desktop


```
pacman -S plasma-desktop plasma-login-manager \
    plasma-pa kscreen xdg-desktop-portal-kde \
    breeze breeze-gtk kde-gtk-config kwallet-pam \
    noto-fonts fastfetch power-profiles-daemon
```

## Networking, firewall and audio


```
pacman -S networkmanager plasma-nm firewalld plasma-firewall python-gobject \
    pipewire pipewire-audio pipewire-pulse wireplumber
```

## Applications


```
pacman -S konsole dolphin ark kate gwenview
pacman -S firefox btop vlc kcalc okular
pacman -S spectacle kinfocenter gnome-disk-utility
```

# Enable the required system services


```
systemctl enable NetworkManager.service firewalld.service plasmalogin.service power-profiles-daemon.service
```


>If you use Bluetooth, add:

```
pacman -S bluez bluez-utils bluedevil
systemctl enable bluetooth.service
```


# Graphics driver

Before rebooting, install the graphics stack that matches your GPU. 

## NVIDIA


```
nvidia-open-dkms nvidia-utils 
```


>Only for **supported Turing and newer NVIDIA GPUs**. Older NVIDIA GPUs require the proprietary `nvidia` driver packages instead. 


## INTEL


```
mesa vulkan-intel
```


## AMD


```
mesa vulkan-radeon
```


## VirtuaL Machine


```
mesa mesa-utils
```



>**Install only the option appropriate for your hardware.**


# Exit the chroot


```
exit
```


# Unmount the installed system


```
umount -R /mnt
```



```
reboot
```


<img width="1018" height="767" alt="Limine-boot-menu" src="https://github.com/user-attachments/assets/048190af-f2ec-42f2-a352-a4d4cce44ee6" />

<img width="1014" height="766" alt="KDE-Login" src="https://github.com/user-attachments/assets/6532d061-5517-4a28-a5d0-4caeb5901ac9" />

<img width="1021" height="765" alt="KInfoCenter" src="https://github.com/user-attachments/assets/c8990c2a-0d73-45de-b24c-0066e5c2ab89" />

<img width="1019" height="765" alt="btop" src="https://github.com/user-attachments/assets/30c77a61-9997-4dc9-b2d5-ef7e2be26336" />



