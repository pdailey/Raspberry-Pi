# Introduction to the Manual
The following outlines the steps I use to setup a Raspberry Pi as a remote server collecting data from bluetooth low energy sensors running Arduino Code. It is mostly a reference for myself, but hopefully it proves useful to others.

The system collects data for a week at a time, offline and without a human operator checking its status, so it is rather robust.

#### Features:
* An external real time clock to keep time without access to a network
* A Node-Red server collecting data every 15 minutes from a BLE sensor array
* Several Cron jobs to process csv data files.

In addition, this is a collection of commands I have found useful WRT the Raspberry Pi.

# Initial Setup
http://blog.self.li/post/63281257339/raspberry-pi-part-1-basic-setup-without-cables

1. Mount the Disk (This is not an exhaustive guide, make sure there are no missing steps here.)
`diskutil list`
`diskutil unmountDisk /dev/disk2`
`sudo dd bs=64 if=/Users/peterdailey/Desktop/R.img of=/dev/disk2`

2. Connect
Check devices on network
`nmap -sn 192.168.0.0/24`
Connect to raspberry pi at home.
`ssh -o StrictHostKeyChecking=no pi@192.168.0.105`

3. Configure the pi
`sudo raspi-config`
Set password, change locale, change timezone

4. Update Applications
`sudo apt-get update && sudo apt-get --assume-yes dist-upgrade`
`sudo apt-get autoremove`

5. Install Essential Apps
`sudo apt-get --assume-yes install vim watchdog git lsof`

6. Install Bluetooth

7. Setup Node Red

8. Setup the RTC

9. Setup Cron

10. Back Up the Pi to Save Progress!

## Install Watchdog
**TODO**


## Install Bluetooth
1. Install the required packages
` sudo apt-get install bluetooth`
`sudo reboot`
2. Show all connected devices and make sure the bluetooth peripheral is recognized.
`lsusb`
If it isn't recognized, reboot.

3. Perform a BLE Scan
Next, scan to see if the Bean is discoverable and within range.
` sudo hcitool lescan `
If you don't see your bean in the list, try turning it off/on again or  replacing the batteries.

4. Make sure that you can use BLE w/o sudo ( http://unix.stackexchange.com/questions/96106/bluetooth-le-scan-as-non-root)
` hcitool lescan `
You should see the same output as running with sudo. Otherwise, continue to the section below.

###Allow non-root users to use BLE
Assuming you are using Node-Red as an ordinary user (safer than running as root), you will need to give permissions to run BLE to non-root users.

```
sudo usermod -G bluetooth -a <username>    // replace <username> with actual username
cat /etc/group | grep bluetooth    // Check that operation performed successfully (should see: bluetooth:x:111:pi)
sudo reboot // Save changes
```

## Install Node Red
1. Install essential supporting packages for using bluetooth
`sudo apt-get install bluetooth bluez libbluetooth-dev libudev-dev`
2. Install nodejs & npm
`curl -sL https://deb.nodesource.com/setup_4.x | sudo bash -`
`sudo apt-get --assume-yes install -y build-essential python-dev python-rpi.gpio nodejs`
3. Install node-gyp to eliminate most warnings
`sudo npm install -g node-gyp`
4. Install node-red
`sudo npm install -g --unsafe-perm  node-red`


#### Run Node Red on Startup
Use built in systemd capability
```
sudo wget https://raw.githubusercontent.com/node-red/raspbian-deb-package/master/resources/nodered.service -O /lib/systemd/system/nodered.service
sudo wget https://raw.githubusercontent.com/node-red/raspbian-deb-package/master/resources/node-red-start -O /usr/bin/node-red-start
sudo wget https://raw.githubusercontent.com/node-red/raspbian-deb-package/master/resources/node-red-stop -O /usr/bin/node-red-stop
sudo chmod +x /usr/bin/node-red-st*
sudo systemctl daemon-reload
```

To then enable Node-RED to run automatically at every boot and upon crashes
`sudo systemctl enable nodered.service`

run the following commands and look for the hidden hints (may need to highlight what looks like blank text)
```
pm2 save
pm2 startup systemd
```
Visit http://nodered.org/docs/getting-started/running.html for potential error in this step. Will end up searching for file that looks like this:
` sudo find / -type f -name "pm2-init.sh"`

#### Install Light Blue Bean Nodes
```
npm install gyp//
mkdir -p ~/.node-red/node_modules
npm install --prefix ~/.node-red node-red-contrib-bean
```

**Misc**
- Run Node Red Manually
node-red-pi --max-old-space-size=128

- View cvs logs
cat /usr/lib/node_modules/node-red/TempLightLog.csv

- List globally installed packages
`npm list -g --depth=0`

## Backup Pi
// http://sysmatt.blogspot.com/2014/08/backup-restore-customize-and-clone-your.html
//Requirements:
apt-get install dosfstools
apt-get install rsync


// Short Verison
// Install Sysmatt's Scripts at http://sysmatt.blogspot.com/2014/08/backup-restore-customize-and-clone-your.html
// to both archive and extract scripts, modify the tar command to preserve permissions
tar cvpfz
tar xvpfz

// Making a backup archived
mkdir -p /home/pi/backups
sysmatt.rpi.backup.gtar

// Restoring / Cloning
dmesg

// !! Connect the backup disk AFTER booting
sudo su
dmesg
// looking for sad or sdb (currently shows up as sdb)
parted /dev/sdb
mklabel msdos
y
mkpart primary fat16 1MiB 64MB
mkpart primary ext4 64MB -1s
print
quit

mkfs.vfat /dev/sdb1
mkfs.ext4 -j /dev/sdb2

mkdir /tmp/newpi
mount /dev/sdb2 /tmp/newpi
mkdir /tmp/newpi/boot
mount /dev/sdb1 /tmp/newpi/boot
df -h

// Added permissions (essential for running BLE scan without root)
rsync -avp --one-file-system / /boot /tmp/newpi/

umount /tmp/newpi/boot
umount /tmp/newpi

## BETA: Setup USB Flash Drive
`sudo apt-get install hfsplus hfsutils hfsprogs`
// On Mac
sudo diskutil list
sudo diskutil disableJournal Drive

// On Pi
sudo blkid

## BETA: udev
TODO
// http://www.reactivated.net/writing_udev_rules.html#about
udev provides out-of-the-box persistent naming for storage devices in the /dev/disk directory. To view the persistent names which have been created for your storage hardware, you can use the following command:
`ls -lR /dev/disk`

When deciding how to name a device and which additional actions to perform, udev reads a series of rules files. These files are kept in the /etc/udev/rules.d directory, and they all must have the .rules suffix.

Default udev rules are stored in /etc/udev/rules.d/50-udev.rules. You may find it interesting to look over this file - it includes a few examples, and then some default rules proving a devfs-style /dev layout. However, you should not write rules into this file directly.

Files in /etc/udev/rules.d/ are parsed in lexical order, and in some circumstances, the order in which rules are parsed is important. In general, you want your own rules to be parsed before the defaults, so I suggest you create a file at /etc/udev/rules.d/10-local.rules and write all your rules into this file.


## Applications
- Update Applications
`sudo apt-get update && sudo apt-get dist-upgrade && sudo apt-get autoremove`
- Remove Applications
`sudo apt-get remove --purge libreoffice\*`
`sudo apt-get clean && sudo apt-get autoremove`
- List Installed Applications
`sudo dpkg -l | more` OR `sudo dpkg -l | less`
- List by date added
`cat /var/log/dpkg.log | grep " install "`




## Installing Cron
http://www.devils-heaven.com/raspberry-pi-cron-jobs/

Cron is already installed on the Jessie Distro. To set it up, follow the steps below.
NOTE: ALWAYS add a line break to your Cron Job

#### Common Tasks
- View Crontab
`crontab -l`

- Check if Cron is running
`service cron status`

#### Setup Cron tasks
Change to directory where cron jobs are stored
`cd /etc/cron.d`
`ls`

Add the following 2 jobs:
TODO: Fix numbering

1. Rotate Logs
`sudo vim rotateLogs`
Write the following lines:
```
59 23 * * * pi sh /home/pi/Scripts/rotateLogs.sh
[LINE BREAK]
```

2. Backup to USB
`sudo vim backupToUSB`
Write the following lines:
```
@midnight root rsync -a /home/pi/Logs /media/Drive
[LINE BREAK]
```

#### File: rotateLogs.sh
```
#!/bin/bash
# Date/Time
TODAY=`date '+%Y-%m-%d'`;

# Source and Destination directories
# Note lack of trailing / makes output easier to discern
src=/usr/lib/node_modules/node-red
dst=/home/pi/Logs

# Copy files in source directory
# TODO After testing, change cp to mv
# TODO Save to USB and /log directory
for f in $src/*.csv
do
        filename=$(basename "$f")
        extension="${filename##*.}"
        filename="${filename%.*}"

        out=$filename-$TODAY.$extension
        # echo "Processing: "$filename""
        # echo "Renaming and moving to desktop: "$out""

        # Remove double quotes from strings
        sed -i 's/\"//g' "$f"
        # Copy to the destination
        mv "$f" "$dst"/"$out"
done

```
## Commands
```
// make a file executable
chmod u+x somefile.sh

 // Repeat a command in terminal every x seconds
                            you don't have to start running ls -ltr again and again, just use watch -n 5 "ls -ltr" and it will run the command every 5 seconds (or any other value by replacing 5 with what you want)

// Sudo the previous command
sudo !!

// restart the raspberry pi
sudo reboot

// Sync folders
rsync -avz -e ssh pi@192.168.0.104:Logs /Users/peterdailey/Desktop/Logs

// close ssh connection
exit

// get current time/date
date

//ntp won't set the date/time if the current clock is off by more than 1000 seconds.
sudo ntpd -q -g
//overrides that restriction.

// halt a program/process
<CONTROL>-C

// Turn off the PI
sudo halt -p
```

## POWER SAVING
- Disable HDMI
Actually a terrible idea, don't do this ever
`sudo vim /etc/rc.local`
Add the line
`/usr/bin/tvservice -o`

## Adding RTC
Pi has Dallas DS3231 I2C RTC chip, but DS1307 commands work and are the standard.

1. Setup I2C
`sudo apt-get install python-smbus i2c-tools`
`sudo raspi-config`
\> Advanced Options > I2C > Yes, enable and yes load by default
`sudo reboot`

2. Connect the RTC to the pins shown in the picture and see that it is recognized.
`sudo i2cdetect -y 1`

3. Remove the module blacklist entry so it can be loaded on boot
`sudo sed -i 's/blacklist i2c-bcm2708/#blacklist i2c-bcm2708/' /etc/modprobe.d/raspi-blacklist.conf`

4. Load the module now
`sudo modprobe i2c-bcm2708`

5. Notify Linux of the Dallas RTC device
`echo ds1307 0x68 | sudo tee /sys/class/i2c-adapter/i2c-1/new_device`

6. Test whether Linux can see our RTC module.
`sudo hwclock`
Read the time off the RTC
`sudo hwclock -r`
Write system time to the RTC
`sudo hwclock -w`

7. Read from the RTC on Startup
`sudo vim /etc/modules`
Add the line to end of file:
`rtc-ds1307`

8. Next you will need to add the DS1307 device creation at boot
`sudo vim /etc/rc.local`
Add the following lines just before exit 0:
```
echo ds1307 0x68 > /sys/class/i2c-adapter/i2c-1/new_device
sudo hwclock -s
date
```

#### Prevent RTC from Drifting
To ensure that the NTP-maintained system clock is periodically saved to the hardware RTC to stop it drifting, I have created in /etc/cron.daily a file called hwRTC. This script updates the RTC from the system clock daily.

`sudo vim /etc/cron.daily/hwRTC`

```
#!/bin/sh
/sbin/hwclock --systohc -D --noadjfile --utc > /tmp/hwClock.log 2>&1
```

Grant the necessary permisisons to run the file from Cron:
`sudo chmod 755 /etc/cron.daily/hwRTCi`
Check that it worked:
`ls -al hwRTC`
should output something like:
> -rwxr-xr-x 1 root root 79 Jun 24 20:17 hwRTC



# Pi LEDs
Red PWR LED
- on if power OK
- flashes (or goes out) if the power drops below about 4.63V
Green ACT LED
- steady on if no SD card during boot
- irregular flashes for SD card access
Ethernet Socket Left LED (yellow)
- on 100-Mbps connection
- off 10-Mbps connection
Ethernet Socket Right LED (green)
- on if link established
- flashes for port activity
- off if no link is established
