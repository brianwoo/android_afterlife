# Android Afterlife - Turn your Android Phone into a Headless Server with Termux

### The following was tested on a Samsung Galaxy S5 Neo (SM-G903W) with LineageOS 18.1 (Android 11)

#### Challenges:
- Limited internal storage (16GB)
- Files stored under SD Card storage will not be executable

## Root your device
Magisk - [link](https://github.com/topjohnwu/Magisk)
  - Flash Magisk (rename apk to zip) via custom recovery (TWRP)
  - Install Magisk Manager app

## Install Apps
- F-Droid - [link](https://f-droid.org/en/)
- Advanced Charging Controller (ACCA) - [link](https://f-droid.org/en/packages/mattecarra.accapp/)
- Termux - [link](https://f-droid.org/en/packages/com.termux/)
- Termux:Boot - [link](https://f-droid.org/en/packages/com.termux.boot/)

## SD Card for Extra Storage 
- Samsung Galaxy S5 Neo only has 16GB internal storage
- Find a 32GB SD Card and format it from the phone, set it to internal storage (adoptable storage)
- Remove SD Card, insert into computer and dump a copy:
  - `dd if=/dev/sdX of=sdcard_image.img bs=4M status=progress` (Linux)
- Get a larger SD Card (64GB or 128GB), insert into computer and restore the image:
  - `dd if=sdcard_image.img of=/dev/sdX bs=4M status=progress` (Linux)
- 32GB of internal storage should now be available on the phone (partitions 1 and 2), the rest of the space on the SD Card can be used for Termux.
- Create a partition on the remaining space (partition 3) and format it as f2fs:
  - `mkfs.f2fs /dev/sdX3` (Linux)
- Create a user_data directory in the F2FS partition.
  - `mkdir -p /mnt/sdX3/user_data` (Linux)
- Insert 128GB SD Card into phone
- <strong>Note: Do not format the SD card from the phone and resize the partition (#2, it's encrypted), this will screw up the partition. App/data won't be able to move to the partition. Use a smaller SD Card to set it up initially and move the image to a bigger SD card as shown.</strong>
  

## Termux Setup
- Open Termux app
- Update packages
  - `pkg update && pkg upgrade -y`
- Install required packages
  - `pkg install openssh termux-services sudo -y`
- Setup password for ssh
  - `passwd`
- Enable sshd service
  - `sv-enable sshd`  
- Mount the SD Card partition the first time
  ```sh
  /system/xbin/su -c "/system/bin/mount -t f2fs /dev/block/mmcblk1p3 /mnt/media_rw/sdcard_ext"

  # Setup ownership and permissions
  /system/xbin/su -c "/system/bin/chown -R $(id -u):$(id -g) /mnt/media_rw/sdcard_ext/user_data"
  ```
- Create a startup script (startup.sh) for Termux:Boot
  ```sh
  #!/system/bin/sh
  # Disable SELinux enforcement and mount SD card partition (partition 3)
  /system/xbin/su -c "/system/bin/setenforce 0"
  /system/xbin/su -c "/system/bin/mkdir /mnt/media_rw/sdcard_ext"
  /system/xbin/su -c "/system/bin/mount -t f2fs /dev/block/mmcblk1p3 /mnt/media_rw/sdcard_ext"
  /system/xbin/su -c "/system/bin/chmod 755 /mnt/media_rw"
  # Optional: symlink usr to SD Card to save internal storage space
  #/system/bin/ln -sfn /mnt/media_rw/sdcard_ext/user_data/termux_usr /data/data/com.termux/files/usr
  termux-wake-lock
  # Run Termux services at startup
  . $PREFIX/etc/profile
  ```
- Make the script executable
  - `chmod +x $HOME/.termux/boot/startup.sh`
- Reboot the phone to test if the script works and the SD Card partition is mounted correctly.
  - `sudo reboot`


## Connecting via ADB
Setup Port Forwarding
```sh
adb forward tcp:8022 tcp:8022
```
Connect via SSH
```sh
ssh -p 8022 username@localhost
```



## Optional: Moving usr to SD Card to save internal storage space
[Original discussion](https://android.stackexchange.com/questions/228443/can-the-termux-environment-be-put-on-an-external-sd-card/228444#228444)
```sh
cp -r -p /data/data/com.termux/files/usr \
  /mnt/media_rw/sdcard_ext/user_data/termux_usr

# Create a backup of usr and symlink
/system/bin/mv /data/data/com.termux/files/usr /data/data/com.termux/files/usr-old
/system/bin/ln -sfn /mnt/media_rw/sdcard_ext/user_data/termux_usr /data/data/com.termux/files/usr

# if it works, delete usr-old
```


## Optional: Setup MariaDB
Install mariadb package
- `pkg install mariadb -y`

Configure MariaDB's data directory:
```sh
# Unfortunately, mariadb data files cannot reside on SD Card due to permission issues.
# Edit $PREFIX/etc/my.cnf
[mysqld]
bind-address = 0.0.0.0
datadir=/data/data/com.termux/files/home/mysql_data
disable_log_bin

# Edit termux-services mysqld run: 
# $PREFIX/var/service/mysqld

#!/data/data/com.termux/files/usr/bin/sh
exec mariadbd-safe 2>&1
```
Setup mariadb as a service
- `sv-enable mysqld`

## Optional: Proot-Distro (dev)
Install proot-distro
- `pkg install proot-distro -y`

Install Ubuntu distribution
- `proot-distro install ubuntu`
- `proot-distro rename ubuntu dev`

Login to the distribution with bind to user_data on SD Card
- `pd login dev --termux-home --bind /mnt/media_rw/sdcard_ext/user_data:user_data`

