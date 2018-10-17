**Arch Linux Installation - Dell Inspiron 15 7000 series**



1. Download the Arch Linux ISO file from https://www.archlinux.org/download/
2. Create a bootable USB issuing the following command.

```bash
dd bs=4M if=/path/to/archlinux.iso of=/dev/sdx status=progress oflag=sync
```

3. For EFI systems, ensure the USB has been formated to FAT32, otherwise `EFI variables not found` error might show up during installation (EFI will not recognize the USB and you will be forced to boot in Legacy Mode).

   During installation, you can confirm that UEFI mode has been enabled by checking if the following directory exists:

   ```bash
   ls /sys/firmware/efi/efivars
   ```

4. Make sure you are connected to the Internet.

   The following shows how to connect to mobile's phone's Internet Connection:

   - Connect the phone device to the laptop via a USB cable.

   - For Androids, under `Settings / Connections / Mobile Hotspot and Tethering` allow `USB Tethering`

   - Run

     ```bash
     ip link
     ```

     and save the id of the device; normally it looks something like `enp0s20ep0`

   - Use `dhcpcd` to connect to the device

     ```bash
     dhcpcd <device name here>
     ```

   - Make sure the connection has been established by running

     ```bash
     ping google.com
     ```

5. Update the system clock.

   ```bash
   timedatectl set-ntp true
   ```

   Check with:

   ```bash
   timedatectl status
   ```

6. Partition the disks.

   For a basic partition layout use the following structure:

   - EFI System Partition `/dev/sda1` with 300 M or 550 M (recommended) of size, FAT32 recommended.
   - Swap Partition `/dev/sda2` with 2 GB (if more than 6 GB of RAM), or 4 GB otherwise, Swap On.
   - Root Partition `/dev/sda3` with at least 20 GB of memory (all the remaining memory preferable), ext4 formatted.

   Easiest way to format the disks is to use `cfdisk`. Run

   ```bash
   cfdisk /dev/sdX
   ```

   where X specifies the disk you want to use for installation. To view a list of the disks, run:

   ```bash
   fdisk -l
   ```

   Using `cfdisk` make sure the label type is `gpt`. Under `Free Space` click `New`. Set the size for the EFI System Partition. Hit enter. Under `Type` choose `EFI System`.

   Under `Free Space` click `New` again. Set the size for the Swap Partition. Hit enter. Under `Type` choose `Linux Swap`.

   Under `Free Space` click `New` one last time. Use rest of free space as size. Under `Type ` choose `Linux Filesystem`.

   Make sure the partition table is the desired one. Finally, hit `Write` (WARNING: This will erase any previous data on the drives). `Quit`.

   Review the partition table has been created successfully by running

   ```bash
   fdisk -l
   ```

7. Format the partitions with the required file systems. We will use FAT32 for EFI System Partition, EXT4 for the Root Partition and create the Swap Partition.

   ```bash
   mkfs.fat -F32 /dev/sda1
   mkfs.ext4 /dev/sda3
   mkswap /dev/sda2
   ```

8. In order to install Arch, the `/(root)` Partition must be mounted to `/mnt` directory in order to be accessible. Also the Swap Partition needs to be initialized. Issue:

   ```bash
   mount /dev/sda3 /mnt
   ls /mnt # check if mounted
   swapon /dev/sda2
   ```

9. After the partitions have been made accessible, select the closest mirror site to increase the download speed of the system. Run:

   ```bash
   vim /etc/pacman.d/mirrorlist
   ```

   and select the closest mirror and put it on top of the list.

   You can also enable **Arch Multilib** support by uncommenting:

   ```
   [multilib]
   Include = /etc/pacman.d/mirrorlist
   ```

   Synchronize and update database mirrors by running

   ```bash
   pacman -Sy
   ```

10. Next, start the system installation by running:

    ```bash
    pacstrap /mnt base base-devel
    ```

11. After the installation completes, generate **fstab** file by running:

    ```bash
    genfstab -U -p /mnt >> /mnt/etc/fstab
    ```

    Assert the file was created by running:

    ```bash
    vim /mnt/etc/fstab
    ```

12. To further configure the installation you must chroot into `/mnt` by running:

    ```bash
    arch-chroot /mnt
    ```

13. Set a hostname for your system:

    ```bash 
    echo "<hostname here>" > /etc/hostname
    ```

14. Next, configure your system language. Run:

    ```bash
    vim /etc/locale.gen
    ```

    and uncomment the following lines

    ```
    en_US.UTF-8 UTF-8
    en_US ISO-8859-1
    ```

    Now generate the system language layout:

    ```bash
    locale-gen
    echo LANG=en_US.UTF-8 > /etc/locale.conf
    export LANG=en_US.UTF-8
    ```

15. Configure your timezone:

    ```bash
    ls /usr/share/zoneinfo
    ln -s /usr/share/zoneinfo/Europe/Sofia /etc/localtime
    ```

    Also, configure your hardware clock to use UTC:

    ```bash
    hwclock --systohc --utc
    ```

16. Next, setup a password for the root account and create a new user with sudo privileges:

    ```bash
    passwd # set root password
    useradd -mg users -G wheel,storage,power -s /bin/bash <your username here>
    passwd <your username here>
    chage -d 0 <your username here>
    ```

17. Install `sudo ` and grant root privileges to the created user:

    ```bash
    pacman -S sudo
    visudo
    ```

    Add this line to the file:

    `% wheel ALL=(ALL) ALL`

18. Next, install the Boot Loader for Arch to boot up after restart. The default Boot Loader in Arch is represented by the GRUB package. Run:

    ```bash
    pacman -S grub efibootmgr dosfstools os-prober mtools
    mkdir /boot/EFI
    mount /dev/sda1 /boot/EFI   # Mount the EFI FAT32 partition
    grub-install --target=x86_64-efi --bootloader-id=grub_uefi --recheck
    ```

    If `EFI variables not found` error appears, then you booted the USB in Legacy Mode. Look at step 3.

19. Create the GRUB configuration file by running:

    ```bash
    grub-mkconfig -o /boot/grub/grub.cfg
    ```

20. Exit chroot environment and reboot:

    ```bash
    exit
    umount -a
    telinit 6
    ```

    You can safely remove the USB drive.

21. Once logged into the system, it is time to configure the `nvidia` drivers, `xorg` server, and `i3-gaps`.

    Blacklist the `nouveau` driver by editing the following file (create it if it doesn't exist):

    ```bash
    sudo vim /etc/modprobe.d/blacklist.conf
    ```

    and add

    ```
    blacklist nouveau
    blacklist rivafb
    blacklist nvidiafb
    blacklist rivatv
    blacklist nv
    blacklist uvcvideo
    ```

    Reboot might be required for the following command to run:

    ```bash
    lspci | egrep 'VGA|3D'
    ```

    If not rebooted, the command may freeze the system. In that case, just do a hard reset.

22. Take note of the Bus ID for the Nvidia Graphics Card device as shown by `lspci` command above. The Bus ID is the first seven characters of the line, i.e., something like `02:00.0`.

23. Install `xorg` and the `nvidia` drivers by running the following:

    ```bash
    sudo pacman -S xorg xorg-xinit xorg-server xorg-apps mesa xterm
    sudo pacman -S nvidia nvidia-utils xorg-xrandr
    ```

24. Create a file under `/etc/X11/xorg.conf` and add the following:

    ``` 
    Section "Module"
    	Load "modesetting"
    EndSection
    
    Section "Device"
    	Identifier "nvidia"
    	Driver "nvidia"
    	BusID "PCI:x:x:x"
    	Option "AllowEmptyInitialConfiguration"
    EndSection
    ```

    Note that under BusID, you should enter the Bus ID of the Nvidia device as in the following example:

    If the ID is `02:00.0` you should enter `2:0:0`.

25. Create another file under `X11/xorg.conf.d/10-nvidia-conf` and add the following:

    ```bash
    Section "Device"
    	Identifier "Device0"
    	Driver "nvidia"
    	VendorName "NVIDIA Corporation"
    	BusID "PCI:x:x:x"
    EndSection
    ```

26. Check the files under `/usr/share/X11/xorg.conf.d` and move any files that contain nvidia information. I.e.

    ```bash
    sudo mv /usr/share/X11/xorg.conf.d/10-nvidia-drm-outputclass.conf /usr/share/X11/xorg.conf.d/10-nvidia-drm-outputclass.conf.bak  # or any other nvidia file
    ```

    Errors in starting `xorg` using `startx` could be related to wrong configurations of the files under `/usr/share/X11/xorg.conf.d`

27. Install Intel microprocessor stuff:

    ```bash
    sudo pacman -S intel-ucode
    ```

    Regenerate the grub config file:

    ```bash
    sudo grub-mkconfig -o /boot/grub/grub.cfg
    ```

28. Enable Nvidia Persistenced service:

    ```bash
    systemctl enable nvidia-persistenced.service
    ```

29. Remake initcpio

    ```bash
    mkinitcpio -p linux
    reboot
    ```

30. Add tapping option to the touchpad (assuming using `libinput`. First copy the default touchpad config file from `/usr/share/X11/xorg.conf.d/40-libinput.conf` to `/etc/X11/xorg.conf.d/40-libinput.conf`. Open `/etc/X11/xorg.conf.d/40-libinput.conf` and under the touchpad section, add:

    ```bash
    Option "Tapping" "on"
    ```

31. To fix MATLAB `MEvent. Case!` bug, disable horizontal scrolling. If using `libinput` do

    ```bash
    xinput --set-prop <device_id> "libinput Horizontal Scroll Enabled" 0
    ```

32. Fix `Unable to locate theme engine in module path “murrine”`. Simply install `gtk-engine-murrine`.

33. Fix `i2c_hid i2c-ELAN1010:00: i2c_hid_get_input: incomplete report` flooding `journalctl` and `dmesg` need to set `acpi_osi=!` under `GRUB_CMDLINE_LINUX_DEFAULT` on `/etc/default/grub`. I.e.,

    ```bash
    GRUB_CMDLINE_LINUX_DEFAULT="splash rcutree.rcu_idle_gp_delay=1 acpi_osi=!"
    ```

    WARNING: The above may cause `GPU has fallen off the bus` on system suspend.