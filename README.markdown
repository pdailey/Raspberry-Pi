# Initial Setup

1. Mount the Disk

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

5. Install Vim, Watchdog & Git
`sudo apt-get --assume-yes install vim watchdog git lsof`

## Setup USB Flash Drive
sudo apt-get install hfsplus hfsutils hfsprogs
// On Mac
sudo diskutil list
sudo diskutil disableJournal Drive

// On Pi
sudo blkid

## Install nodered
// Install essential supporting packages for using bluetooth
sudo apt-get install bluetooth bluez libbluetooth-dev libudev-dev
// install nodejs & npm
curl -sL https://deb.nodesource.com/setup_4.x | sudo bash -
sudo apt-get --assume-yes install -y build-essential python-dev python-rpi.gpio nodejs
// install node-gyp to eliminate most warnings
sudo npm install -g node-gyp
// install node-red
sudo npm install -g --unsafe-perm  node-red

// List globally installed packages
npm list -g --depth=0



// Backup Pi
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

TODO
// http://www.reactivated.net/writing_udev_rules.html#about
udev provides out-of-the-box persistent naming for storage devices in the /dev/disk directory. To view the persistent names which have been created for your storage hardware, you can use the following command:
  # ls -lR /dev/disk

When deciding how to name a device and which additional actions to perform, udev reads a series of rules files. These files are kept in the /etc/udev/rules.d directory, and they all must have the .rules suffix.
Default udev rules are stored in /etc/udev/rules.d/50-udev.rules. You may find it interesting to look over this file - it includes a few examples, and then some default rules proving a devfs-style /dev layout. However, you should not write rules into this file directly.
Files in /etc/udev/rules.d/ are parsed in lexical order, and in some circumstances, the order in which rules are parsed is important. In general, you want your own rules to be parsed before the defaults, so I suggest you create a file at /etc/udev/rules.d/10-local.rules and write all your rules into this file.


// Update Applications
sudo apt-get update && sudo apt-get dist-upgrade && sudo apt-get autoremove
// Remove Applications
sudo apt-get remove --purge libreoffice*
sudo apt-get clean && sudo apt-get autoremove
// List Installed Applications
sudo dpkg -l | more
// or
sudo dpkg -l | less
// List by date added
cat /var/log/dpkg.log | grep " install "

  ## Node Red
  // Run Node Red
  node-red-pi --max-old-space-size=128

  // Run Node Red on Startup
  // Use built in systemd capability
  sudo wget https://raw.githubusercontent.com/node-red/raspbian-deb-package/master/resources/nodered.service -O /lib/systemd/system/nodered.service
  sudo wget https://raw.githubusercontent.com/node-red/raspbian-deb-package/master/resources/node-red-start -O /usr/bin/node-red-start
  sudo wget https://raw.githubusercontent.com/node-red/raspbian-deb-package/master/resources/node-red-stop -O /usr/bin/node-red-stop
  sudo chmod +x /usr/bin/node-red-st*
  sudo systemctl daemon-reload

  // To then enable Node-RED to run automatically at every boot and upon crashes
  sudo systemctl enable nodered.service


  // run the following commands and look for the hidden hints (may need to highlight what looks like blank text)
  pm2 save
  pm2 startup systemd

  // visit http://nodered.org/docs/getting-started/running.html for potential error in this step. Will end up searching for file that looks like this.
  sudo find / -type f -name "pm2-init.sh"

  // install Light Blue Bean Nodes
  npm install gyp
  mkdir -p ~/.node-red/node_modules
  npm install --prefix ~/.node-red node-red-contrib-bean


  // View cvs logs
  cat /usr/lib/node_modules/node-red/TempLightLog.csv


  ## Bluetooth
  sudo apt-get install bluetooth
  sudo reboot
  // Show all connected devices
  lsusb
  // Perform a BLE Scan (make sure there is a device that can be detected nearby)
  sudo hcitool lescan

  // Allow non-root users to use BLE
  sudo usermod -G bluetooth -a <username>   // replace <username> with actual username
  cat /etc/group | grep bluetooth   // Check that operation performed successfully (should see: bluetooth:x:111:pi)
  sudo reboot
  hcitool lescan

  ## Cron
  http://www.devils-heaven.com/raspberry-pi-cron-jobs/
  // ALWAYS add a line break to your Cron Job

  // View Crontab
  crontab -l
  // Check if Cron is running
  service cron status

  // Change to directory where cron jobs are stored
  cd /etc/cron.d
  ls

  // Add the following 2 jobs
  sudo vim rotateLogs
  // Append the following lines
  59 23 * * * pi sh /home/pi/Scripts/rotateLogs.sh
  [LINE BREAK]

  sudo vim backupToUSB
  @midnight root rsync -a /home/pi/Logs /media/Drive
  [LINE BREAK]

  // remove quotes from a file
  sed -i ’s/\”//g’ file.txt


  // rotateLogs.sh
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

                                                                                  ## Commands
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


                                                                                  ## POWER SAVING
                                                                                  // Disable HDMI
                                                                                  sudo vim /etc/rc.local

                                                                                  // Add the line
                                                                                  /usr/bin/tvservice -o

                                                                                  ## Adding RTC
                                                                                  // (Pi has Dallas DS3231 I2C RTC chip, but DS1307 commands work and are the standard)
                                                                                  // Setup I2C
                                                                                  sudo apt-get install python-smbus i2c-tools

                                                                                  sudo raspi-config
                                                                                  > Advanced Options > I2C > Yes, enable and yes load by default
                                                                                  sudo reboot

                                                                                  sudo i2cdetect -y 1
                                                                                  // Remove the module blacklist entry so it can be loaded on boot
                                                                                  sudo sed -i 's/blacklist i2c-bcm2708/#blacklist i2c-bcm2708/' /etc/modprobe.d/raspi-blacklist.conf

                                                                                  // Load the module now
                                                                                  sudo modprobe i2c-bcm2708

                                                                                  // Notify Linux of the Dallas RTC device
                                                                                  echo ds1307 0x68 | sudo tee /sys/class/i2c-adapter/i2c-1/new_device

                                                                                  // Test whether Linux can see our RTC module.
                                                                                  sudo hwclock

                                                                                  // Read the time off the RTC
                                                                                  sudo hwclock -r
                                                                                  // Write system time to the RTC
                                                                                  sudo hwclock -w

                                                                                  // Read from the RTC on Startup
                                                                                  sudo vim /etc/modules
                                                                                  //Add the line to end of file:
                                                                                  rtc-ds1307

                                                                                  // Next you will need to add the DS1307 device creation at boot
                                                                                  sudo vim /etc/rc.local
                                                                                  // Add the following lines just before exit 0:
                                                                                  echo ds1307 0x68 > /sys/class/i2c-adapter/i2c-1/new_device
                                                                                  sudo hwclock -s
                                                                                  date

                                                                                  // To ensure that the NTP-maintained system clock is periodically saved to the hardware RTC to stop it drifting, I have created in /etc/cron.daily a file called hwRTC. This script updates the RTC from the system clock daily.

                                                                                  sudo vim /etc/cron.daily/hwRTC

                                                                                  #!/bin/sh
                                                                                  /sbin/hwclock --systohc -D --noadjfile --utc > /tmp/hwClock.log 2>&1

                                                                                  sudo chmod 755 /etc/cron.daily/hwRTC
                                                                                  ls -al hwRTC
                                                                                  -rwxr-xr-x 1 root root 79 Jun 24 20:17 hwRTC


                                                                                  /// Initial Setup

                                                                                  http://blog.self.li/post/63281257339/raspberry-pi-part-1-basic-setup-without-cables

                                                                                  diskutil list
                                                                                  diskutil unmountDisk /dev/disk2
                                                                                  sudo dd bs=64 if=/Users/peterdailey/Desktop/R.img of=/dev/disk2


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
