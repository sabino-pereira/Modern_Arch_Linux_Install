Table of Contents 

- [1. Introduction](#1-introduction)
- [2. Pre-Install](#2-pre-install)
- [3. Installation](#3-installation)
- [4. Post-Install](#4-post-install)
- [5. Extras](#5-extras)

---


# 1. Introduction
> I started this project as a personal reference document, but ended up formatting it as a guide since I think it will be helpful to many people out there who want the same type of install. 
I will walk through and explain the rationale behind all the steps from beginning to end, towards installing what is (subjectively) a perfect arch linux install.

>Please note that this IS NOT meant to be a replacement of the official arch installation documentation, it's just how I did it.
If you haven't already, read the [official guide](https://wiki.archlinux.org/title/Installation_guide) and try that out in a VM and understand what is happening. 
Then if you want a system like me, follow this guide for reference.

### Main Features of this install:

1. Full Root Filesystem Encryption + Auto TPM unlocking.
2. BTRFS Filesystem
3. Snapshots and rollbacks using [Snapper](https://wiki.archlinux.org/title/Snapper)
4. Secure Boot
5. Unified Kernel Images
6. Dual Boot Support.
7. Snapshot booting with limine
8. System Maintainence and Best Practises.


### I have tested it on the following hardware:

1. AMD CPU (Ryzen 9 7900x - x86-64)
2. AMD GPU (7900 GRE)
3. Asrock Motherboard with UEFI support.

All the steps mentioned here should regardless work for you on an x86-64 machine. If you have an INTEL CPU and/or a NVIDIA GPU, there are some minor differences, and I will point them out.


> While I will try to explain what I'm doing here and why, this is not a beginner friendly guide. You need to have an understanding of the core concepts of linux and know your way around the terminal.


A lot of the stuff I will talk about here, was hugely inspired by other people in the community and their work. Special Credits to:

1. [This guide by Walian which was the main motivator.](https://walian.co.uk/arch-install-with-secure-boot-btrfs-tpm2-luks-encryption-unified-kernel-images.html)
2. [This guide based on fedora by Madhu@sysguides.](https://sysguides.com/install-fedora-with-snapshot-and-rollback-support)
3. [And offcourse, the Arch Wiki.](https://wiki.archlinux.org/title/Main_page)

Finally, I am  by no means an expert, so I welcome all feedback on this. Lets start:

---
---

# 2. Pre-Install
This is all the preparation you need to do before attempting the install.

### 1. Download the arch iso

Either use the BitTorrent download (needs a bit-torrent client installed on your machine) or choose your nearest **https** mirror from the [Arch Download Page](https://archlinux.org/download). 

<br><br>


### 2. Verify the downloaded Image (Optional but recommended)

From the same download page, download the PGP ISO Singature. If you have gnupg installed, run the below command from the same directory where both the signature and iso files are located:
```
gpg --keyserver-options auto-key-retrieve --verify archlinux-<version>-x86_64.iso.sig archlinux-<version>-x86_64.iso
```	

Where archlinux-\<version\>-x86_64.iso.sig is the signature file, and the final option is the iso itself.

<br><br>


### 3. Create an Installation Medium

You need a usb flash drive that can be formmated to hold the installation image.

On windows, use a tool like [Rufus](https://rufus.ie/en/) ~~or[Balena Etcher](https://etcher.balena.io/)~~ . On linux, [KDE ISO Image Write](https://apps.kde.org/isoimagewriter/) or [Gnome Disk Utility](https://gitlab.gnome.org/GNOME/gnome-disk-utility) are some  good Gui applications to do this. Many [more methods](https://wiki.archlinux.org/title/USB_flash_installation_medium) are documented on the wiki. <br>
	
<details style='background-color: #3f50b5;'>
<summary>Ventoy</summary>

You can alternatively use a software like Ventoy, which can be installed to your usb flash drive and then all you need to do is place the iso on the drive to boot from it. 

More importantly, it allows you to have multiple iso images on the same disk. I personally use and love it. You can check out more about Ventoy on it's [Github page](https://github.com/ventoy/Ventoy).
</details>

<br><br>

### 4. Identify the Installation Target

If you're like me, and are installing linux on a separate drive, while already having windows on another drive, you need to check and make sure to correctly identify the drive. In linux, you can list all your drives using the command 'lsblk'. It labels SATA drives as sda1,sda2,... and nvme drives as nvme0n1, nvme1n1, etc. To avoid nuking your windows install, make a note of the correct drive. You can identify drives by their labels, existing partition layouts, storage capacity, etc.

<br><br>

---
---

# 3. Installation

In this section, we will boot off the usb flash drive with the Arch ISO created in the previous section and use it to install a minimal Arch Linux with all the base utilities.

### 1. Boot from the ISO

Plug in your usb with the Arch ISO, reboot to your PC's Motherboard settings (lookup how to for your model, generally it's by pressing the **F2**, **F10** or **Del** key during boot), and in the boot priority, set the usb as the first one to boot from. 

> If secure boot is enabled, you will have to disable it.
	
>Further in this guide, I will show you how to generate custom secure boot keys. We will sign our boot files with those keys, so that we can then re-enable secure boot.

> As a bonus step, I will also show how you can sign the arch iso with the same secure boot keys, so that if you ever need to do any maintainence on your system using the iso, you don't have to turn off secure boot.


<br><br>


### 2. Internet Connectivity

If you have a wired ethernet connection, just check for connectivity using the **ping** command shown below and proceed to step 3.<br>

If you're on wifi, run the below commands sequentially to connect to your network:

```
iwctl									-> Starts the iwctl utility, and puts your inside a new prompt
device list								-> List all the connection options available to you
device wlan0 set-property Powered on    -> Turns on wlan0 (wifi module)
station wlan0 scan						-> Scans for wifi networks
station wlan0 get-networks				-> Lists results of the scan
station wlan0 connect <wifi-name>     	-> Connects to provided wifi name
<Enter you wifi password>				-> Enter password for wifi
exit									-> Exit the Prompt
```

> Here wlan0 is the name of my wifi device as output by *device list*, and \<wifi-name> will be substituted by the name of my Wifi Connection.


Check connectivity using:
```
ping -c 5 archlinux.org
```


<br><br>


### 3. Check for disks
Run  the below command to list all the disks available on your system:

```
lsblk
```

Find the name of your target disk, in my case it is nvme1n1.

If you don't have a nvme SSD, your disk may be listed as sda,sdb, etc.

To avoid any mistakes during partitioning, especially in a dual boot environment, save the disk name in a variable using:

```
DISK=/dev/nvme1n1
echo $DISK
```

<br><br>

### 4. Partition your disk
In this step, I will create a new GPT Partition table on the disk, and then create 2 partitions - a 4GB EFI partiton (also referred to as the ESP) and a Root partition on the remaining space.


> I am seriously debating the size of the EFI partition. While 512Mb has been more than enough for me is the past, I have seen people recommend more. 
I have a 1 Tb SSD on which I will be installing arch, so it's not a big deal for me, and as I plan to use this system long term, I don't want to have to deal with resizing the partition later, so I am making it 4 GB. 
If you want to have it as 512Mb, use +512M instead of +4G in the 6th command below:

Type in the following commands in order (wherever it says default, just press the Enter key). Also note that whatever changes you do are not applied till you type the last command (w), so don't worry if you mess up anything, you can exit the prompt and restart.

````
fdisk $DISK        ---> Starts the fdisk partition tool on the selected disk and puts you into it's prompt
g				   ---> Command to create a new gpt disk label
n				   ---> Command to create a new partition
*default*          ---> Asks for partition number, press Enter, as default selection is 1, which is correct.
*default*		   ---> Asks for first sector, press Enter, as default is start of the Disk Space.
+4G                ---> Asks for last sector, which sets size of partition 1, this command will make it 4 GB.
n                  ---> Command to create a new partition
*default*          ---> Asks for partition number, press Enter, as default selection is 2, which is correct.
*default*          ---> Asks for first sector, press Enter, as default is start of the Disk Space after previous partition.
*default*          ---> Asks for last sector, press Enter, as default is all of the remaining Disk Space.
t                  ---> Command to change type of partition
1				   ---> Select Partition 1
1                  ---> Set partition 1's type as 1 (EFI System)
t   			   ---> Command to change type of partition
2				   ---> Select Partition 2
23				   ---> Set partition 2's type as 23 (Linux root x86-64)
w                  ---> Command to save all the changes we have done till now
````

For the first partition, I set the parition type as EFI, and for the second one as Linux Root (x86_64) instead of the default (Linux Filesystem).


This setting of partition type, while technically not necessary, is essential, as this will associate a standard [Partition Type GUID](https://unix.stackexchange.com/questions/121176/whats-the-difference-between-the-partition-guid-code-and-partition-unique-guid) with it.


This is then used by the systemd initramfs hook to recognize our root partition, decrypt, and mount it automatically without a crypttab or fstab file, using a feature called [Discoverable Partitions](https://www.freedesktop.org/software/systemd/man/latest/systemd-gpt-auto-generator.html). 

Verify your partition type GUID with:

```
lsblk -p -o NAME,PARTTYPE
```

Your Root Filesystem ${DISK}2 should have a guid of 4f68bce3-e8cd-4db1-96e7-fbcaf984b709. 

The EFI parition ${DISK}1 should have the guid of c12a7328-f81f-11d2-ba4b-00a0c93ec93b

Run **lsblk** to verify your partition sizes.

If your disk name was sda,sdb, you can skip this step. If it was a nvme drive like me, update the DISK variable as shown below. This is because partitons in nvme are named as nvme1n1p1, nvme1n1p2 while other are named as sda1,sda2,
	
```
DISK=/dev/nvme1n1p
```

<br><br>


### 5. Encryption (Optional but recommended)
Now we will encrypt the root partition with LUKS2.

> When we use luks to encrypt our drive with a password or a key, what it does is create a header. This header is what actually encrypts and decrypts the drive. 
The password is saved in a 'keyslot', which unlocks only the header. This is different that plain encryption, where the password is used to encrypt the entire disk. This allows a greater flexibility when creating and managing passwords and keys. We can also have multiple 'keyslots' (i.e. backup keys) created that can unlock the header. [Read More](https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup)

> One downside of this approach is that if our header ever gets corrupted, we lose the ability to unlock our entire drive. To account for this, we will also backup the header further in the guide, so that it can be restored if the need arises.
	
<br>

Run the following commands to encrypt the Root Partition:

```
cryptsetup -v luksFormat --type luks2 ${DISK}2
cryptsetup open --type luks --allow-discards --perf-no_read_workqueue --perf-no_write_workqueue --persistent ${DISK}2 root
```
<br>

In the first command we formatted our disk with luks2, it will autogenerate the header and ask you to provide a password to unlock. Make this password strong, or better yet use a [passphrase](https://www.techtarget.com/searchsecurity/definition/passphrase).
	
In the second command, we open our encrypted drive, and give it the name of 'root'. From here on, our root partition isn't /dev/sda2 or /dev/nvme1n1p2, but rather it's /dev/mapper/root. 


> For simplicity, you can think of this as a drive inside another drive. The outer drive is the encryption container, and all our content will be on the inner drive.

<br>

I also used some additional options while opening the encrypted drive. They do the following:

1. The *--allow-discards* enables [TRIM](https://wiki.archlinux.org/title/Solid_state_drive#TRIM)  support for a SSD. [Read More](https://wiki.archlinux.org/title/Dm-crypt/Specialties#Discard/TRIM_support_for_solid_state_drives_(SSD))
	
2. The *--perf-no_read_workqueue* and *--perf-no_write_workqueue* options increase performance for SSD's [Read More](https://wiki.archlinux.org/title/Dm-crypt/Specialties#Disable_workqueue_for_increased_solid_state_drive_(SSD)_performance)

3. The *--persistent* option saves these options to the header, so they are always used by default in the future.

<br>
You can check the flags that your drive is opened with using:
	
<br>

```
cryptsetup luksDump ${DISK}2 | grep Flags
```



<br><br>


### 6. Format your Partitions

>If you encrypted your root paritition, your new root partition is /dev/mapper/root. If you didn't, it going to be ${DISK}2. Substitute it in the appropritate places. I'll be using /dev/mapper/root.

Now I will format the efi partition as  Fat32 and the Root partition as BTRFS filesystem.

```
mkfs.vfat -F32 -n ESP ${DISK}1
mkfs.btrfs -L root /dev/mapper/root
```
	
Check if the partitons are correctly created:
```
blkid -o list
```
		
<br><br>

### 7. Btrfs Subvolumes
> This section is a lot more subjective to the type of installation you prefer. Basically, since we are on a btrfs filesystem, and will be using the rollback functionality on the root subvolume, you can choose which folders on your root won't be rolled back in that transaction. <br>
They can have their own rollback logic created. To do this, they need to be mounted as separate subvolumes. Choosing which folders should be subvolumes has no correct answer (though there are wrong answers).
	
	
The archinstall script formats the following directories as subvolumes, using the '@' naming scheme:

/, /home, /var/log, /var/cache/pacman/pkg.

<br>

[OpenSuse Recommends the following Subvolumes](https://en.opensuse.org/SDB:BTRFS), using the directory name as the subvolume name:

/home, /opt, /root, /srv, /tmp, /usr/local, /var.

<br>

This is the subvolume layout I personally use, and which has worked well for me:

| Subvolume Name    | Mount Point | Purpose |
| ----------------- | ----------- | ------- |
| @ | / | The root folder, which is also a separate subvolume below the BTRFS Root volume, and which we will rollback in case of any issues. |
| @home | /home |  Home Folder where all your Data/Games/Configurations will reside. |
| @mozilla | /home/$USER/.mozilla | The directory where your firefox data is stored. If you ever rollback your home directory, this will prevent any potential loss of browser data. |
| @ssh | /home/$USER/.ssh | Same as above, to protect any ssh keys/configs you have. |
| @games| /home/$USER/Games | All my games will be installed here to avoid issues when rolling back home. |
| @opt | /opt |  This is where third party applications are installed.|
| @var | /var |  Where all temporary files/logs/cache is stored. |
| @snapshots | /.snapshots |  This subvolume will store snapshots of our @ subvolume (Managed via Snapper). |
| @home-snapshots | /home/.snapshots | This subvolume will store snapshots of our @home subvolume (Managed via Snapper). |

>Steam by default uses folders other than the **/home/\$USER/Games** to download games, but this can very easily be changed in Steam settings. Apps like Heroic Games Launcher by default use the /home/$USER/Games folder, so no issues there.

>I am making the whole of **/var** into one subvolume. This is not recommended by arch, as pacman stores it's cache in the */var/cache/pacman* directory. 
But I am going to configure pacman cache to be in the */tmp* directory, you can read the rationale behind this in this [reddit post](https://www.reddit.com/r/archlinux/comments/1hgbl1k/what_is_varcachepackagepkg_and_why_is_it_so_large/). 
But essentially, I am preferring to not save any pacman cache, so it doesnt matter to me.

<br>

Run the below commands to mount the root subvolume and create all the other subvolumes:

```
mount /dev/mapper/root /mnt
cd /mnt
btrfs subvolume create {@,@home,@mozilla,@ssh,@games,@var,@opt,@snapshots,@home-snapshots}
btrfs subvolume list /mnt
```

The final commands lists all the subvolumes with their ID.
Get the id of the '@' subvolume (eg. 256) and set it as default for the root:

```
btrfs subvolume set-default 256 /mnt
```


Optionally, if you use Thunderbird, Chrome, or Gnupg you can also create the below subvolumes to preserve their data during rollbacks, similar to @mozilla above.
| Subvolume Name    | Mount Point |
| ----------------- | ----------- | 
| @thunderbird | /home/$USER/.thunderbird |
| @chrome | /home/$USER/.config/google-chrome |
| @gnupg | /home/$USER/.gnupg |



>Btrfs subvolumes can also be created in the future on an operational system, if you need it. 
		

<br><br>

### 8. Mount Options

Mount all the subvolumes you created using the below commands:

```
umount /mnt
mount -o defaults,noatime,space_cache=v2,ssd,compress-force=zstd:1,commit=120,subvol=@ /dev/mapper/root /mnt
mkdir /mnt/{home,opt,var}
mount -o subvol=@home /dev/mapper/root /mnt/home
mount -o subvol=@opt /dev/mapper/root /mnt/opt
mount -o subvol=@var /dev/mapper/root /mnt/var
```

[Read more about various BTRFS mount Options](https://btrfs.readthedocs.io/en/stable/Administration.html#mount-options). If you want to change any of the mount options, you can do so. It's not going to break anything.

> Once a subvolume is mounted with a set of options, all other subvoumes follow the same options, regardless of how you mount them. 

> This is fine mostly as I need the same options everywhere, except in the @var directory, where I want to have the [Nodatacow](https://www.reddit.com/r/btrfs/comments/n6slx3/what_is_the_advantage_of_nodatacowdisabling_cow/) mount option. 
This is because I do not want compression and datasums on the contents on the var folder, as it's suppoed to contain non-essential data, and disabling CoW can have some performance benefits.
So I will set the attribute +C on the var directory, which accomplishes the same thing.

Run this command to disbale CoW in the @var subvolume

```
chattr +C /mnt/var
```

I will mount the @mozilla, @ssh, @games, @snapshots and @home-snapshots subvolumes, later in the guide.

<br>


Mount the efi partition using:

```
mkdir -p /mnt/efi
mount -o defaults,fmask=0077,dmask=0077 ${DISK}1 /mnt/efi
```


<br><br>

### 9. Update Mirrors and Pacstrap

Before we download and install the necessary packages, let us update our Pacman mirrors, for the best download speed, by running the reflector command. Replace $COUNTRY with your country name (e.g. India):

```
reflector --country $COUNTRY --age 24 --latest 10 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
```

Now, I will install the base required packages for an Arch Linux install using the pacstrap command on /mnt where the system root is mounted. Press enter after running these commands:

#### <mark style='background-color:#3f50b5'> If you have an intel cpu, replace 'amd-ucode' with 'intel-ucode' in the second command. </mark>

	
```
pacman -Sy archlinux-keyring
pacstrap -K /mnt base base-devel linux linux-firmware linux-headers amd-ucode cryptsetup btrfs-progs efibootmgr limine dosfstools networkmanager sudo openssh vim reflector rsync arch-install-scripts xdg-user-dirs git
```

> Some of these packages are mandatory, others are essential. You can lookup what they do in case you're curious.


<br><br>

### 10. Chroot into the install and do basic setup

>Chroot or Change-Root makes it so that the system behaves as if you are the root user of the new install. This way you can setup some basic things as the Root user without needing to reference /mnt as the root.

```
arch-chroot /mnt
```

<br>

1. Set the system timezone, Update the system time and Setup a NTP server for Clock Accuracy using the below commands (replace /Asia/Kolkata with your respective timezone)
		
	```
	ln -sf /usr/share/zoneinfo/Asia/Kolkata /etc/localtime
	timedatectl set-ntp true
	systemctl enable systemd-timesyncd
	```

	[Read this if you are dual-booting windows](https://wiki.archlinux.org/title/System_time#UTC_in_Microsoft_Windows)

2. Locale

	```
	vim /etc/locale.gen
	```

	Uncomment your desired locale by removing the # at the start. Also uncomment en_US.UTF-8 ([Why?](https://wiki.archlinux.org/title/Steam#Installation))
	
	Save the file and run:

	```
	locale-gen
	```

	Then run the following command and provide appropriate inputs when asked (go with defaults if in doubt):

	```
	systemd-firstboot --prompt
	```

3. Pacman (Optional but recommended)

	1. Improve pacman defaults:
			
		```
		vim /etc/pacman.conf
		```
		Under misc option, make sure the following lines are added/uncommented:

		---
		<pre style="background-color:green;">
		Color
		ParallelDownloads = 5
		ILoveCandy
		</pre>

		---

		<br>

		Save and Exit.

		<br>

	2. Changing the Pacman cache directory location:

		```
		vim /etc/pacman.conf
		```

		Remove the '#' from the line starting with #CacheDir, and make it look like:

		---
		<pre style="background-color:green;"> 
		CacheDir     = /tmp/cache-pacman/pkg/ 
		</pre>
		
		---


<br><br>
	
### 11. User Management

1. Set Root Password
	```
	passwd
	```

2. Create a new user with preferred username (sabino in this case) and set a password.

	```
	useradd -G wheel,video,audio,optical,storage,users -m sabino
	passwd sabino
	```

3. Allow the new user to run sudo commands with a password
	
	```
	visudo
	```

	Uncomment by removing the '#' of the line starting with '# %wheel', below the comment saying "Uncomment to allow members of group wheel to execute any command"

	---
	<pre style="background-color: green;">
	%wheel ALL=(ALL:ALL) ALL 
	</pre>

	---

	Fun easter egg:
	Add the following line to the **/etc/sudoers** file using *visudo* to see a funny message everytime you enter a wrong password:

	---
	<pre style="background-color: green;">
	Defaults insults
	</pre>
	
	---

	But don't have too much fun with it, if you enter a wrong password 3 times, you will not be able to use the sudo command for some time (10 mins normally). 


4. Mount 3 of the remaining 5 subvolumes, (replace sabino with whatever username you chose):
	```
	mkdir /home/sabino/{.mozilla,.ssh,Games}
 	chown -R sabino:sabino /home/sabino/*
	mount -o subvol=@mozilla /dev/mapper/root /home/sabino/.mozilla
	mount -o subvol=@ssh /dev/mapper/root /home/sabino/.ssh
	mount -o subvol=@games /dev/mapper/root /home/sabino/Games
	chown -R sabino:sabino /home/sabino/*
	```


<br><br>

### 12. Generate fstab

> fstab is a file referenced by the system during boot to mount your drives/partitions to the correct location. 
Since we have already mounted our drives, we will use the **genfstab** utility to output the fstab file with the options we chose earlier, and save it, so that in the future these options are used by default.

```
mkdir /etc                       -> It might already exist
touch /etc/fstab				 -> It might already exist
genfstab -U / >> /etc/fstab
```

<br><br>

### 13. Setting up SWAP (optional but recommended)
For Swap, I will use Zswap along with a swap file. Earlier I used to use Zram, but I've decided that Zswap is the better option.


1. Creating the swapfile:
	```
	btrfs filesystem mkswapfile --size 16G --uuid clear /var/swapfile
	swapon /var/swapfile
	```

	Then add the below line in fstab:
	```
	/var/swapfile	none	swap	defaults	0	0
	```

2. To allow hibernation (suspend to disk), it needs to be setup in the kernel parameters.
	First, we need to find the location of our swapfile to pass to the kernel. This can be found by using:
	```
	btrfs inspect-internal map-swapfile -r /var/swapfile
	```

	Note down the number output by the above command and add it to /etc/kernel/cmdline as:
	```
	echo "resume=/dev/mapper/root resume_offset=<number>" >> /etc/kernel/cmdline
	```

3. The stable linux kernel has Zswap enabled by default. To configure additinal settings and make them persist, we can add them to the kernel cmdline (/etc/kernel/cmdline):
	```
	echo "zswap.enabled=1 zswap.shrinker_enabled=1 zswap.compressor=lz4 zswap.max_pool_percent=30" >> /etc/kernel/cmdline
	```

<br>

[Suspend and Hibernate](https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#) - Read more about how these things work.

I had some troubles waking my system back up after suspending it (It seems to be motherboard specific). On reading the wiki, I can across this [solution](https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#PC_will_not_wake_from_sleep_on_A520I_and_B550I_motherboards), which worked perfectly.

<br><br>


### 14. Generate Unified Kernel Images (Optional but HIGHLY recommended, there's no downside to doing this, the benefits are many.)
[Unified Kernel Images](https://wiki.archlinux.org/title/Unified_kernel_image) is a concept of combining the kernel, microcode, and other binaries needed during boot, and generating a single efi file to boot from. This helps in maintainence also helps with secure boot management.

While this may sound difficult, it's been made very easy by various tools. I will use [mkinitcpio](https://wiki.archlinux.org/title/Mkinitcpio) in this guide. By just changing a few Configuration options, it will be set up to generate the UKI's automatically when any of the component files are updated.

Run the below commands:
```
echo "quiet rw" >> /etc/kernel/cmdline
mkdir -p /efi/EFI/Linux
vim /etc/mkinitcpio.conf
```
	
Edit the line starting with *HOOKS* in mkinitcpio.conf to look like this:

---
<pre style="background-color:green"> 
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole sd-encrypt block filesystems fsck) 
</pre>

---

Now we need to edit the linux kernel preset file. (If you have installed additional kernels, you need to update their preset files as well.)

```
vim /etc/mkinitcpio.d/linux.preset
```
Comment the line starting with default_image and fallback_image and uncomment the lines starting with ALL_config, default_uki, default_options and fallback_uki.
Make the file look like this:

---
<mark style="background-color:green"> 
All_config="/etc/mkinitcpio.conf" <br>
ALL_kver="/boot/vmlinuz-linux" <br>
PRESETS=('default', 'fallback') <br>
default_uki="/efi/EFI/Linux/arch-linux.efi" <br>
default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp" <br>
fallback_uki="/efi/EFI/Linux/arch-linux-fallback.efi" <br>
fallback_options="-S autodetect"
</mark>

---


Run below command to generate the unified kernel images:
```
mkinitcpio -P
```

If you get an error specifying a particular file is missing during the above command, create that file, and then rerun the above command.
eg. I got told that the file **/etc/vsoncole.conf** was missing. So I created it and reran the command:
```
touch /etc/vconsole.conf
mkinitcpio -P
```

<br><br>

### 15. Enable services

Using the below commands, I will enable networking using systemd-resolve for dns resolution and NetworkManager to manage networks. An alternative to NetworkManager is systemd-networkd, but I will mask it, as I prefer NetworkManager. I will also enable reflector to update pacman mirrors weekly, and sshd to enable ssh.

```
systemctl enable systemd-resolved NetworkManager sshd reflector.timer
systemctl mask systemd-networkd
```
For reflector, you should edit the file /etc/xdg/reflector/reflector.conf, and specify option specific to you. eg.:

```
sudo vim /etc/xdg/reflector/reflector.conf
```

My file looks like this:

---
<mark style="background-color:green">
--save /etc/pacman.d/mirrorlist<br>
--protocol https<br>
--country India<br>
--latest 10<br>
--sort rate<br>
--age 24
</mark>

---

[Read more about you can configure Reflector](https://ostechnix.com/retrieve-latest-mirror-list-using-reflector-arch-linux/).



### 16. Bootloader Installation

> I will be installing Limine, relying on our Unified Kernel Image and systemd's discoverable partitions to automatically recognize and try to decrypt the root partition. If you decide to use grub, you have to so [some more setup in the crypttab file.](https://bbs.archlinux.org/viewtopic.php?id=290148)

In this step, I will install the Limine bootloader on the /efi partition and configure it to boot our Unified Kernel Image. Then I will reboot to the Motherboard settings:

```
mkdir -p /efi/EFI/limine
cp /usr/share/limine/BOOTX64.EFI /efi/EFI/limine/
```

Then to create an entry for limine in NVRAM:
```
efibootmgr --create --disk $DISK --part 1 --label "Limine Bootloader Arch" --loader "\EFI\limine\BOOTX64.EFI" --unicode
```

We also need to so some basic configuration for limine:
```
vim /efi/EFI/limine/limine.conf
```

---
<pre style="background-color:green">
timeout: 5

/Arch Linux
    protocol: efi
    path: boot():/EFI/Linux/arch-linux.efi

/Arch Linux (fallback)
    protocol: efi
    path: boot():/EFI/Linux/arch-linux-fallback.efi
</pre>
---


Exit chroot and reboot to firmware setup:

```
sync
exit
systemctl reboot --firmware-setup
```


In the Firmware setup menu, select the entry titled "Limine Bootloader Arch" as the **First Priority Entry** and save the settings.

On boot, limine should show up and you will have the option to select your kernel (Arch vs Arch fallback). Selected Arch and the boot sequence will begin. You will be asked to enter the disk encryption password, post which you can login and use the your new Arch System.


**Congrats, you have a vanilla arch linux system installed.**

<br>

---
---

# 4. Post-Install
> While we have a working system now, we are missing a lot of utilities which make it actually usuable. In this section, I will walk through installing and configuring them.

Boot into your system, and login as your user.

Before we start, the system needs to connected to the network (wifi). If you have ethernet, you can skip this. For wifi users, who are using Network Manager, run:
```
nmcli radio wifi on
nmcli dev wifi list
sudo nmcli dev wifi connect <wifi-name> --ask
```

substitue \<wifi-name> for wifi name and enter password to connect.


### 1. Update Pacman and install some essential packages:

1. Enable the multilib repository [Read more](https://wiki.archlinux.org/title/Official_repositories#)

    I will be enabling the Multilib repo (because Steam isn't available on the default repo as it's a 32 bit application).
		
	```
	vim /etc/pacman.conf
	```
		
	Uncomment the below lines by removing the '#'
	
	---
	<pre style="background-color: green;">
	[multilib]
	Include = /etc/pacman.d/mirrorlist
	</pre>

	---

	

	Then run:

	```
	sudo pacman -Sy
	```

2. Installing Drivers and Codecs
		
	Generic:
	```
	sudo pacman -S ffmpeg gstreamer mesa 
	```

	Amd GPU Specific:
	```
	sudo pacman -S vulkan-radeon lib32-mesa
	```

	#### <mark style='background-color:rgb(46, 152, 154)'> Nvidia GPU:</mark>
	>I don't have an Nvidia GPU, so you have to test this part yourself, refer to this [link](https://wiki.archlinux.org/title/NVIDIA).
	
3. AUR Helper (Optional but recommended)
	>I will install [yay](https://github.com/jguer/yay) to help me download and install AUR packages.
				
	```
	sudo pacman -S --needed base-devel
	git clone https://aur.archlinux.org/yay.git
	cd yay
	makepkg -si
	```

	This command might take a long time and you will have to enter your password a couple of time. Be patient. Once it has finished successfully, you can cleanup the downloaded folder:
	```
	rm -rf yay
	```
	
	>After this step, you will be able to use the command *yay -S \<package-name>* to install a package from the [AUR](https://aur.archlinux.org/). Be very careful when installing packages from there.

4. Audio
		
	```
	sudo pacman -S pipewire pipewire-pulse wireplumber
	```


5. Utilities (Optional but HIGHLY recommended)
			
	```
	sudo pacman -S locate udisks2 unzip man-db man-pages wget htop
	```


6. Flatpaks (Optional but HIGHLY recommended)
	
	```
	sudo pacman -S flatpak
	flatpak install flatseal
	```

	>Flatseal is a GUI app that can help to manage the permissions of flatpaks installed. It can also help to enable Wayland support for apps like Obsidian, in case you want to use fractional scaling.

	You can search for any other applications you need on [Flathub](https://flathub.org/).

 
<br><br>

### 2. Install rEFInd (Optional but recommended)

Refind is a boot manager. Simply put, when starting your PC, it will allow you to choose between Windows and Linux. It can also be themed to make the boot process look pretty.

```
sudo pacman -S refind
sudo refind-install
```

>Once you run the second command, refind should set itself as the primary boot option. If it doesn't, you can change it in the motherboard settings. But because of this, you should now see multiple options during boot, including the ability to choose between windows and arch. For Arch, I recommend using the limine option, instead of the kernel option. You can hide any options you don't want using the 'Delete' Key.

<details>
<summary>Why do I need both rEFInd and limine? </summary>

- Refind has GUI support.
- Multi OS Support (Windows, Linux, and MacOS)
- It is customizable and can be [made to look very pretty](https://refind-themes-collection.netlify.app/).
- I want the main linux bootloader to do just that, and not have other fancy features and changes in configuration, improving realiability.
</details>


<br><br>

### 3. Setup Secure Boot (Optional but recommended)

>I won't explain what secure boot is, but it's really not necessary for a linux distro, period. If you're dual booting Windows, it's needed in the newer versions (11) and also some games using kernel level anticheat like Valorant need it. There aren't any downsides to having it enabled, and since it's super easy these days to setup on linux, so why not?

We will generate our own secure boot keys, enroll them in our Motherboard firmware, and sign our kernel modules and other efi binaries with them. There's an amazing tool known as [sbctl](https://github.com/Foxboron/sbctl) which does most of the heavy lifting for us.

```
sudo pacman -S sbctl
```

Once sbctl is downloaded, you need to make sure your motherboard is in setup mode, so that you can enroll your keys into it. This is done typically by clearing out the existing secure boot keys in the motherboard settings. Look up how to do this for your brand of motherboard.

<mark style="background-color: red" > Many motherboards have the ability to restore the keys that you removed (I can confirm this for Asrock). Also if you ever update the firmware of the motherboard, you might have to enroll your own keys again. </mark>

Check if motherboard is in Setup mode from the output of:

```
sbctl status
```

Now we will generate our keys, enroll them into the motherboard(along with Microsoft keys for windows comapatibility).

These keys are stored in the '/var/lib/sbctl/keys' directory. It's not a bad idea to back them up.

```
sudo sbctl create-keys
sudo sbctl enroll-keys -m
sudo sbctl status
```

After confirming the enrollment of keys, we will sign our kernel, bootloader and boot manager(refind) files with our keys. 

If any of these images get updated, we will also instruct sbctl to sign them again automatically.

The *sbctl verify* command lists all files that need to be signed and checks whether they are signed or not. 

We will then use *sbctl sign -s \<filename>* to sign them.

```
sudo sbctl verify
sudo sbctl sign -s /efi/EFI/linux/arch-linux.efi
sudo sbctl sign -s /efi/EFI/linux/arch-linux-fallback.efi
sudo sbctl sign -s /efi/EFI/limine/BOOTX64.EFI
sudo sbctl sign -s /efi/EFI/refind/drivers_x64/btrfs_x64.efi
sudo sbctl sign -s /efi/EFI/refind/refind_x64.efi
```

You might have to sign more files, if you have installed additional kernels. Use the *sbctl verify* command to check if any more files need signing.

Now go to your motherboard's firmware setup and enable secure boot. Check if it boots properly or not.

<br><br>

### 4. Improve Encryption Setup (Optional but Recommended)

1. Backup LUKS Headers

	I have already explained previously why you might want to backup your encrypted drive's Luks header, which stores data about and controls your drive decryption.
	To backup the header, run:

	```
	sudo cryptsetup luksHeaderBackup /dev/gpt-auto-root-luks --header-backup-file luks-header-backup.img
	```

	This will generate a file 'luks-header-backup.img' in your current directory. 
	
	Save this in another drive, in the cloud (depending on your threat model), or into a password manager like bitwarden. If your header ever gets corrupted, you can boot from another iso, and restore it using:

	```
	cryptsetup luksHeaderRestore /dev/device --header-backup-file /path-to/luks-header-backup.img
	```

2. Manage Keyslots:

	While we normally use a passord to unlock the luks header, it can support multiple unlocking options, one of which is a keyfile. 
	
	In this optional step, I will generate a keyfile with random data, and enroll it into the header. This can then be used to unlock the drive.

	Generate a strong key into a file named my-encryption-key in your current folder:

	```
	openssl genrsa -out my-encryption-key 4096
	```

	Enroll this key into your root drive:
	```
	sudo cryptsetup luksAddKey /dev/gpt-auto-root-luks my-encryption-key
	```

	You can use this key to unlock the drive (assuming you are booted from an iso) using:
	```
	cryptsetup open --type luks /dev/your_disk_partition root --key-file my-encryption-key
	```

	<br>

	View all keyslots in your drive's luks header:
	```
	cryptsetup luksDump /dev/gpt-auto-root-luks
	```

	or
	```
	sudo systemd-cryptenroll /dev/gpt-auto-root-luks
	```

	The keyslots are numbered starting from 0, in the order that you added them.

	You can delete an exsting keyslot using:
	```
	cryptsetup luksKillSlot /dev/gpt-auto-root-luks \<keyslot-number>
	```

	Be careful not to remove all the keys as this might render your drive unlockable. (You can recover from such an accident if you have the Luks Header Backed up)


3. Setup auto unlocking via TPM [Read More](https://0pointer.net/blog/unlocking-luks2-volumes-with-tpm2-fido2-pkcs11-security-hardware-on-systemd-248.html)

	We can enroll our TPM2 chip (present in most modern PC's) on the LUKS volume with the help of [Systemd-cryptenroll](https://wiki.archlinux.org/title/Systemd-cryptenroll) to have the TPM2 then automatically unlock the drive during boot.

	This unlocking mechanism also be bound the PCR's on the TPM (basically it checks some stuff like software version,etc. on the pc before unlocking. This is an additional security measure.). We can also add a pin to this.
	
	Check TPM support on your PC using:

	```
	systemd-analyze has-tpm2	
	```

	If you see an output like tpm0, then a tpm device exists and you are good to do the next steps.
		
	Choose which PCR's to use. A pcr is a register on the TPM that can measure specific values. In this case, I will be using PCRs 0, You can see the full list [Here](https://wiki.archlinux.org/title/Trusted_Platform_Module#Accessing_PCR_registers)

	I will use the following PCR's:
	
	| PCR Number | Name | What it measures | What will cause change |
	| --- | --- | --- | --- |
	| 0 | platform-code | System Firmware | Changes on firmware update of the motherboard |
	| 1 | platform-config | System Firmware configuration | Changes if you change any hardware components like RAM, CPU, etc, |
	| 7 | secure-boot-policy | State of secure boot | Changes if secure boot is turned on/off or keys are updated. |

	This makes it so that if I upgrade my Motherboard firmware, hardware components or turn off secure boot the TPM won't be able to auto unlock my drive. You can use more PCR's if you wish to.

	The following enrolls the TPM into the Luks header, while also binding it to the PCR's 0, 1, and 7:
	
	<mark style="backgroung-color: red;"> Note: Run the below command, after enabling secure boot, otherwise it will get bound to 'Secure Boot Disabled' state of PCR 7 </mark>

	```
	sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+1+7  /dev/gpt-auto-root-luks
	```

	Verify the enrollment with:
	```
	sudo systemd-cryptenroll /dev/gpt-auto-root-luks
	```

	You should see an entry with "tpm2", along with any other passphrases or keys you enrolled in the Luks drive. Try to reboot your pc and this time you should not have to enter the encryption password.

<br><br>

### 5. Security  (Optional but Recommended)

To start with this section, first read the [Arch wiki article on security](https://wiki.archlinux.org/title/Security).

Another good [Article on hardening arch linux](https://vez.mrsk.me/linux-hardening).

There's a lot you can do in terms of hardening your system, here are some of the simple things:

1. Choose a secure password
   
2. SSH: 

	If you have a SSH server running, and exposed to the internet, it's not a bad idea to follow [this guide](https://www.ibm.com/docs/en/aspera-fasp-proxy/1.4?topic=appendices-securing-your-ssh-server) that will help you to:
   1. Disable SSH Root login
   2. Change the default ssh port
   3. Disable password login and only use public keys

3. Firewall - UFW

   Here are some commands to use ufw and setup a basic firewall:
   
   ```
   sudo pacman -S ufw
   sudo ufw limit ssh
   sudo ufw default deny incoming
   sudo ufw enable
   ```

   >This should be enough for a basic desktop. If you run any servers/application, you may have to whitelist those ports.
   Note that if you run any docker containers, it bypasses ufw rules, so keep it in mind. [Read More](https://docs.docker.com/engine/network/packet-filtering-firewalls/#docker-and-ufw)

4. Motherboard password

	If your motherboard settings are changed, anyone can disable secure boot/ load another os. It's highly recommended to password protect your motherboard settings. Most modern motherboards give you the option. Lookup how to do it for your particular brand. Also make sure to note down this password, if you forget it, it's gonna be a pain to recover.

5. AppArmor/Firejail/Bubblewrap:

	While these are recommneded to be used, I have not personally used them, so I can't give any recommendations right now. If you think you need them, read up on it and use it. I will also use it and maybe update the guide with my learnings.

<br><br>


### 6. Snapper and snapshots
> Alright, all the efforts we put into btrfs and subvolumes, will help us now.

By now, you must be aware of the btrfs snapshot feature. There is a tool developed by OpenSuse know as [snapper](https://github.com/openSUSE/snapper) which is a helper application for this purpose. 
Using snapper, we will create a config for our root subvolume (@) that contains most of our system data, and another for the home subvolume(@home), which contains most of our user data. 

Then we can setup snapper to create and manager snapshots of them, to safeguard and if necessary, rollback this data (Remember, the other subvolumes we created are exempt from this). 
Snapper can also do automatic snapshots on a schedule, and there's a pacman hook known as snap-pac that can create snapshots whenever we use pacman to install/upgrade our system.
There's also a gui application btrfs-assistant that can help with managing snapshots. 

Install them using:

```
sudo pacman -S snapper snap-pac
paru -S btrfs-assistant
```

Run the below commands in order to:
1. Create Snapper Configurations for the root and home subvolumes.
	```
	sudo snapper -c root create-config /
	sudo snapper -c home create-config /home
	sudo snapper list-configs
	```
	After this, delete the /.snapshots, and /home/.snapshots  subvolumes, and instead mount @snapshots and @home-snapshots in their place:
	```
	sudo btrfs subvolume delete /.snapshots
	sudo btrfs subvolume delete /home/.snapshots
	sudo mkdir /.snapshots
	sudo mkdir /home/.snapshots
	sudo mount -o subvol=@snapshots /dev/mapper/root /.snapshots
	sudo mount -o subvol=@home-snapshots /dev/mapper/root /home/.snapshots
	```

	Add these entries to your fstab.
	```
	sudo genfstab -U / | grep snapshots >> /etc/fstab
	```

2. Enable your user to work with snapshots without sudo privileges:
	```
	sudo snapper -c root set-config ALLOW_USERS=$USER SYNC_ACL=yes
	sudo snapper -c home set-config ALLOW_USERS=$USER SYNC_ACL=yes
	```

3. Disable Indexing of snapshots to improve performance.
	```
	echo 'PRUNENAMES = ".snapshots"' | sudo tee -a /etc/updatedb.conf
	```

	>Since we installed snap-pac, from now on, whenever we install any update/package using pacman, it will create a pre-post snapshots for that transaction.
	So if something get's messed up during an update, you can rollback the changes to the pre snapshot and be good to go. (remember to run *mkinitcpio -P* again after a rollback if the update included any kernel/microcode changes).


4. Auto Snapshots Timer and Cleanup
	By default, both snapshot configs will have automatic creation of periodic snapshtos enabled. You can control this behaviour. I will disable auto snapshots for home, and start the associated services:
	```
	sudo snapper -c home set-config TIMELINE_CREATE=no
	sudo systemctl enable --now snapper-timeline.timer
	sudo systemctl enable --now snapper-cleanup.timer
	```

	The default timeline is once per day, you can control this behaviour by editing the file */etc/snapper/configs/root*. You can specify the number and age of snapshots for creation and cleanup.

	You can also take a manual snapshot of root at any point using:
	```
	snapper -c root create --description <Description>
	```

	I will generally take a manual snapshot after I have setup my system, and then anytime later when I feel I have a stable system with a lot more changes than the pervious snapshot.

	If you instead want to take a pre-post snapshot after running a command, use:
	```
	snapper -c root create --command <command-to-run>
	```

5. Working with Snapper:

	1. List all snapshots
		```
		snapper -c <config> ls
		```

		If config is not specified, it's root by default eg.:
		```
		snapper ls
		```

	2. Create a new Snapshot Manually
		```
		snapper -c <config> create --description <desc>
		```

		eg.:
		```
		snapper -c root create --description "A new snapshot"
		```

	3. Delete a snapshot
		```
		snapper -c <config> delete <snapshot-number>
		```

		You can also delete a range of snapshots using:

		```
		snapper -c <config> delete <snapshot-number-start>-<snapshot-number-end>
		```

	4. Undo changes made between snapshots
		If you have 2 snapshots 2 and 3, and you want to go back to snapshot 12 (revert changes made during creation of 13), you can do this using:
		```
		sudo snapper undochange 2..3
		```

   You can refer to this [link](https://sysguides.com/install-fedora-with-snapshot-and-rollback-support#6-7-snapper-tests) for some more practice on how to use snapshots.


> At this point we have a system with a lot of modern features and a good base install, while also having the capability to rollback in case of any issues. 

> You can stop using the guide now and make most of the further decisions on your own as to what type of Desktop Environment, Apps, Theming and Workflow you prefer. I will document my own rice and dotfiles in [Another Repo](https://github.com/sabi-31/My_Arch-Linux_Dotfiles) (Hyprland).

---
---

# 5. Extras

> I will document some more stuff here, which might be interesting to you, but also can be safely ignored if you wish so.

1. SSD Maintainence

	You need to periodically run the TRIM service on your SSD, which improves it's life. [Read More about TRIM](https://wiki.archlinux.org/title/Solid_state_drive#TRIM)

	Check if you target SSD supports TRIM using
	```
	lsblk --discard
	```

	Check the values of DISC-GRAN (discard granularity) and DISC-MAX (discard max bytes) columns. Non-zero values indicate TRIM support.

	I will use periodic trim (on a timer).

	```
	sudo systemctl enable fstrim.timer
	```

	This will perform the TRIM operation weekly on all supported SSD's.

	To manuall run fstrim, type:
	```
	sudo systemctl start fstrim.service
	```
<br><br>

2. General Maintainence
	
	[Read the full article from the wiki](https://wiki.archlinux.org/title/System_maintenance)

	1. Check for failed systemd services
		```
		systemctl --failed
		```
	2. Check Boot times
		Running the below command will give you the total boot time, broken down into individual components like firmware, bootloader, kernel, etc.
		```
		systemd-analyze
  		```

		If you want a per-service breakdown of boot times, run:
		```
  		systemd-analyze blame
  		```
		
  
 	4. Check all the logs generated during boot (latest to oldest). Errors will be listed in red:
		```
		sudo journalctl -exb
		```
	5. System Upgrades
		Upgrade the system regularly for the latest patches and bugfixes. If you are using paru, you can also run the below command to update all pacman and AUR packages.
		
		```
		sudo pacman -Syu
		```
		
		OR

		```
		paru
		```

		Offcourse, read up on release notes and any news before upgrading.

	6. Stay upto date with arch news
		Due to the nature of arch being a rolling release, I would highly recommend regularly reading up on the lastest news, announcements, and other events.

		1. Subscribe to their mailing list [from here](https://lists.archlinux.org/mailman3/lists/)

		2. Visit the [Homepage](https://archlinux.org/)

		3. Use a tool like [Informant](https://github.com/bradford-smith94/informant) which will stop you from upgrading if there are any unread news items. Install using:
			```
			paru -S informant
			```

			```
			informant check -> Auto run with pacman, fetches latest news
			informant list -> Gives a summary of latest news
			informant read -> Marks unread news as read, post which you can upgrade using pacman.
			```

  <br><br> 

3.  Help the community

	You can send an anonymized list of all installed packages, the architecture and the mirror you are using as feedback to the arch project.

	This is done by using [pkgstats](https://wiki.archlinux.org/title/Pkgstats).
	```
	sudo pacman -S pkgstats
	```

	After this, it will automatically enable a timer that will run the service weekly.

<br><br>



4. Firmware Upgrades

	Using [fwupd](https://github.com/fwupd/fwupd), you can upgrade the firmware of your hardware devices. 
	This downloads updates from [Linux Vendor Firmware Service](https://fwupd.org/). 
	Manufactures submit updates there directly so it's safe and reliable to use this service to update your firmware.

	Run a check for all devices detected:
	```
	fwupdmgr get-devices
	```

	Download metadata regarding firmware
	```
	fwupdmgr refresh
	```

	List devices for which upgrades are available
	```
	fwupdmgr get-updates
	```

	Apply available updates
	```
	fwupdmgr update
	```


---
---

# BONUS 

### Sign and Repack an Arch iso with your own keys [Read More](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#ISO_repacking)

Once you have enabled secure boot after registering your own keys, you cannot boot from the same iso you used to install arch. 
You might want to do this in case you want to fix any potential issues with your system. 

>You can offocurse disable secure boot to boot from the iso, but I prefer not to.

All of the below commands are run assuming you're in a folder which has the **archlinux.iso** in it. Also you need to have setup sbctl and generated and registered you own keys, as documented above.

1. Install the necessary packages
	```
	sudo pacman -S libisoburn mtools
	```

2. Extract files from the iso
	```
	osirrox -indev archlinux.iso -extract_boot_images ./ -cpx /arch/boot/x86_64/vmlinuz-linux /EFI/BOOT/BOOTx64.EFI /EFI/BOOT/BOOTIA32.EFI /shellx64.efi /shellia32.efi ./
	```
3. The extracted file are read-only be default, make the editable using:
	```
	chmod +w BOOTx64.EFI BOOTIA32.EFI shellx64.efi shellia32.efi vmlinuz-linux
	```
4. Sign the below files using your own keys generated by SBCTL
	```
	sudo sbctl sign BOOTx64.EFI
	sudo sbctl sign BOOTIA32.EFI
	sudo sbctl sign shellx64.efi
	sudo sbctl sign shellia32.efi
	sudo sbctl sign vmlinuz-linux
	```
5. Copy the signed binaries to eltorito_img2_uefi.img
	```
	mcopy -D oO -i eltorito_img2_uefi.img vmlinuz-linux ::/arch/boot/x86_64/vmlinuz-linux
	mcopy -D oO -i eltorito_img2_uefi.img BOOTx64.EFI BOOTIA32.EFI ::/EFI/BOOT/
	mcopy -D oO -i eltorito_img2_uefi.img shellx64.efi shellia32.efi ::/
	```
6. Repack and create the signed ISO
	```
	xorriso -indev archlinux.iso -outdev archlinux-signed.iso -map vmlinuz-linux /arch/boot/x86_64/vmlinuz-linux -map_l ./ /EFI/BOOT/ BOOTx64.EFI BOOTIA32.EFI -- -map_l ./ / shellx64.efi shellia32.efi -- -boot_image any replay -append_partition 2 0xef eltorito_img2_uefi.img
	```

>You should now have a archlinux-signed.iso file that you can use on your system, as it is signed with your registered keys.
