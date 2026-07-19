# Android Afterlife - Turn your Android Phone into a Headless Server with Termux

### The following was tested on a Samsung Galaxy S5 Neo (SM-G903W) with LineageOS 18.1 (Android 11)

#### Challenges with Android (un-modified):
- Charging cannot be capped to certain percentages when plugged in (Older Android version)
- Limited internal storage (16GB)
- Files stored under SD Card storage are not executable

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
- Find a bigger SD Card and format it from the phone, set it to external storage
- Remove the SD Card and put it in Linux. Create a an image file (ext_disk.img)
  - truncate -s 10G ext_disk.img
  - mkfs.ext4 -O ^metadata_csum,^64bit,^orphan_file,^extra_isize ext_disk.img  (need these extra flags for S5 Neo)

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
  /system/xbin/su -c "/system/bin/mount -t exfat /dev/block/mmcblk1p1 /mnt/media_rw/global_sd"

  # Setup ownership and permissions
  /system/xbin/su -c "/system/bin/chown -R $(id -u):$(id -g) /mnt/media_rw/global_sd"
  ```
- Create a startup script (startup.sh) for Termux:Boot
  ```sh
  #!/system/bin/sh
  # Disable SELinux enforcement and mount SD card partition
  /system/xbin/su -c "/system/bin/setenforce 0"
  # Detach any stale loop controllers before mount globally without using Android vold
  /system/xbin/su -t 1 -c "/system/bin/losetup -D" 2>/dev/null
  /system/xbin/su -t 1 -c "/system/bin/mkdir -p /mnt/media_rw/global_sd"
  /system/xbin/su -t 1 -c "/system/bin/mount -t vfat /dev/block/mmcblk1p1 /mnt/media_rw/global_sd" 2>/dev/null || \
  /system/xbin/su -t 1 -c "/system/bin/mount -t exfat /dev/block/mmcblk1p1 /mnt/media_rw/global_sd"

  # Mount the image as a loopback
  LOOP_DEV=$(/system/xbin/su -t 1 -c "/system/bin/losetup -f")
  /system/xbin/su -t 1 -c "/system/bin/losetup $LOOP_DEV /mnt/media_rw/global_sd/ext_disk.img"
  /system/xbin/su -t 1 -c "/system/bin/mount -t ext4 -o rw,exec $LOOP_DEV /data/data/com.termux/files/home/ext_disk"
  
  # Mount swap
  /system/xbin/su -t 1 -c "/system/bin/swapon /data/data/com.termux/files/home/swapfile"
  
  # start services
  termux-wake-lock
  . $PREFIX/etc/profile
  
  sleep 5

  # Optional.... start Caddy, or other services....
  caddy start --config /data/data/com.termux/files/home/ext_disk/Projects/Caddy/Caddyfile
  
  # Disable some services to save memory and to run as a headless mode (check with procrank)
  sleep 5
  # To grep all the service's names
  # grep -h '^service ' /system/etc/init/*.rc /vendor/etc/init/*.rc /init.rc
  /system/xbin/su -c "/system/bin/stop bootanim"
  /system/xbin/su -c "/system/bin/stop media"
  /system/xbin/su -c "/system/bin/stop statsd"
  /system/xbin/su -c "/system/bin/stop vendor.drm-hal-1-0"
  /system/xbin/su -c "/system/bin/pm hide com.android.systemui"
  /system/xbin/su -c "/system/bin/pm hide com.android.launcher3"
  /system/xbin/su -c "/system/bin/pm disable com.android.inputmethod.latin"
  /system/xbin/su -c "/system/bin/pm disable com.android.nfc"
  /system/xbin/su -c "/system/bin/pm disable org.lineageos.audiofx"
  /system/xbin/su -c "/system/bin/pm disable org.lineageos.settings.doze"
  /system/xbin/su -c "/system/bin/pm disable org.calyxos.backup.contacts"
  /system/xbin/su -c "/system/bin/pm disable com.android.localtransport"
  # Run this once to disable seedvault backup manager: bmgr enable false
  /system/xbin/su -c "/system/bin/pm disable com.stevesoltys.seedvault"
  /system/xbin/su -c "/system/bin/settings put global max_cached_processes 1"
  /system/xbin/su -c "/system/bin/settings put system screen_brightness 0"
  /system/xbin/su -c "/system/bin/sendevent /dev/input/event0 1 116 1 && /system/bin/sendevent /dev/input/event0 0 0 0 && /system/bin/sendevent /dev/input/event0 1 116 0 && /system/bin/sendevent /dev/input/event0 0 0 0"

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
By default adb forward can only forward a device's port to a local port on the computer. To allow access of port 8022 from anywhere in the LAN:
```sh
sudo sysctl net.ipv4.conf.eth0.route_localnet=1
sudo iptables -t nat -A PREROUTING -p tcp --dport 8022 -j DNAT --to-destination 127.0.0.1:8022

# Persist the iptables rules
sudo apt install iptables-persistent
sudo dpkg-reconfigure iptables-persistent
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

Configure MariaDB's data directory (This might not be needed anymore because we are using the external SD Card, than adaptive):
```sh
# Unfortunately, mariadb data files cannot reside on SD Card due to permission issues.
# To workaround this issue, we can create a an image file on the SD Card and use this image partition as our MariaDB data files.
# To create an image file (do this on the phone):
truncate -s 10G /mnt/media_rw/sdcard_ext/mysql_data.img

# Format this image as ext4 (for newer devices)
mkfs.ext4 /mnt/media_rw/sdcard_ext/mysql_data.img

# Alternatively, format this image as ext4 (for older 32-bit devices, like the S5 Neo)
mkfs.ext4 -O ^metadata_csum,^64bit,^orphan_file,^extra_isize /mnt/media_rw/sdcard_ext/mysql_data.img

# To mount the mysql_data.img as a loop device, I had to split the mount into the following lines to work:
LOOP_DEV=$(/system/xbin/su -t 1 -c "/system/bin/losetup -f")
/system/xbin/su -t 1 -c "/system/bin/losetup $LOOP_DEV /mnt/media_rw/sdcard_ext/mysql_data.img"
/system/xbin/su -t 1 -c "/system/bin/mount -t ext4 -o rw,exec $LOOP_DEV /data/data/com.termux/files/home/mysql_data_dir"
/system/xbin/su -c "chown -R u0_111:u0_111 /data/data/com.termux/files/home/mysql_data_dir"

# Edit $PREFIX/etc/my.cnf
[mysqld]
bind-address = 0.0.0.0
datadir=/data/data/com.termux/files/home/mysql_data_dir/mysql_data
disable_log_bin
innodb_buffer_pool_size = 64M
performance_schema = OFF

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


## Reboot / Shutdown
If the phone is refused to reboot / shutdown, kill processes and umount the partition
```sh
pkill mariadbd
sudo umount /data/user/0/com.termux/files/home/mysql_data_dir
sudo umount /data_mirror/data_ce/null/0/com.termux/files/home/mysql_data_dir
sudo umount /data/data/com.termux/files/home/mysql_data_dir
```

