# Introduction
The following outlines the steps I use to setup a Raspberry Pi as a remote server collecting data from bluetooth low energy sensors running Arduino Code. It is mostly a reference for myself, but hopefully it proves useful to others.

The system collects data for a week at a time, offline and without a human operator checking its status, so it is rather robust.

#### Features:
* An external real time clock to keep time without access to a network
* A Node-Red server collecting data every 15 minutes from a BLE sensor array
* Several Cron jobs to process csv data files.

In addition, at the bottom of this page is a collection of commands I have found useful in working with the Raspberry Pi.

# Initial Setup
[Basic headless setup for secure pi.](http://blog.self.li/post/63281257339/raspberry-pi-part-1-basic-setup-without-cables)

- Install the operating system image using the [linked guide](https://www.raspberrypi.org/documentation/installation/installing-images/mac.md).
- Connect: Connect the Pi to the local network by plugging an ethernet cable from the router into the board and then turning it on.

```bash
# Check devices on network
nmap -sn 192.168.0.0/24

# Connect to Raspberry Pi at home. This is a little bit of a hack solution...
ssh -o StrictHostKeyChecking=no pi@192.168.0.105
# default password is "raspberry"
```

- Configure the Pi

```bash
sudo raspi-config
# Set password, change locale, change timezone

# Restart to save preferences
sudo reboot
```

- Update Applications
```bash
sudo apt-get update && sudo apt-get -y dist-upgrade
sudo apt-get autoremove
```

- Install Essential Apps. Your list of essential apps may be different.
```bash
sudo apt-get --assume-yes install vim watchdog git lsof
```

- Setup Watchdog [use above reference]

- Setup Bluetooth [see below]

- Setup Node Red [see below]

- Setup the RTC [see below]

- Setup Cron [see below]

- Backup the Pi:
[Follow the steps in this tutorial.]( http://sysmatt.blogspot.com/2014/08/backup-restore-customize-and-clone-your.html)


## Setup Bluetooth
- Plug in the bluetooth USB on the Pi. Install the required packages:
```bash
sudo apt-get install bluetooth
sudo reboot
```

- Show all connected devices and make sure the bluetooth USB is recognized.
``` bash
lsusb
```
If it isn't recognized, reboot again (Hey, sometimes that actually works).

- Perform a BLE Scan
Next, scan to see if the Bean is discoverable and within range.
``` bash
sudo hcitool lescan
```
If you don't see your bean in the list, try turning it off/on again or  replacing the batteries. It can also be helpful to use a different BLE device, as the beans can be finicky.

- Make sure that you can use BLE w/o sudo ( http://unix.stackexchange.com/questions/96106/bluetooth-le-scan-as-non-root)
```bash
hcitool lescan
```
You should see the same output as running with sudo. If so, you are done. Otherwise, continue to the section below.

- Allow non-root users to use BLE
Assuming you are using Node-Red as an ordinary user (safer than running as root), you will need to give permissions to run BLE to non-root users.

```bash
# replace <username> with actual username
sudo usermod -G bluetooth -a <username>

# Check that operation performed successfully (should see: bluetooth:x:111:pi)
cat /etc/group | grep bluetooth
sudo reboot
```

## Setup Node Red
- Install essential supporting packages for using bluetooth.
```bash
sudo apt-get install bluetooth bluez libbluetooth-dev libudev-dev
curl -sL https://deb.nodesource.com/setup_4.x | sudo bash -
sudo apt-get --assume-yes install -y build-essential python-dev python-rpi.gpio nodejs
```

- Install node-gyp to eliminate most warnings
```bash
sudo npm install -g node-gyp
```

- Install node-red
```bash
sudo npm install -g --unsafe-perm  node-red
```



#### Run Node Red on Startup
Use built in systemd capability
```bash
sudo wget https://raw.githubusercontent.com/node-red/raspbian-deb-package/master/resources/nodered.service -O /lib/systemd/system/nodered.service
sudo wget https://raw.githubusercontent.com/node-red/raspbian-deb-package/master/resources/node-red-start -O /usr/bin/node-red-start
sudo wget https://raw.githubusercontent.com/node-red/raspbian-deb-package/master/resources/node-red-stop -O /usr/bin/node-red-stop
sudo chmod +x /usr/bin/node-red-st*
sudo systemctl daemon-reload
```

To then enable Node-RED to run automatically at every boot and upon crashes
`sudo systemctl enable nodered.service`

Run the following commands and look for the hidden hints (may need to highlight what looks like blank text)
```
pm2 save
pm2 startup systemd
```
Visit http://nodered.org/docs/getting-started/running.html for potential error in this step. Will end up searching for file that looks like this:
` sudo find / -type f -name "pm2-init.sh"`

#### Install Light Blue Bean Nodes
[Source](https://github.com/PunchThrough/node-red-contrib-bean)
```bash
npm install gyp
mkdir -p ~/.node-red/node_modules

# Allow non-root users to run node-red. This grants the node binary cap_net_raw privileges, so it can start/stop BLE advertising.
sudo setcap cap_net_raw+eip $(eval readlink -f `which node`)

npm install --prefix ~/.node-red node-red-contrib-bean
```

#### Create Flows for node-RED
- Go to the following address, substituting the IP address of the Pi:
<http://192.168.0.104:1880/#>

- Import the node-RED flow files
```
> Hamburger Menu > Import > From Clipboard
```

- Paste the following
```js
[{"id":"14230480.ebdcfc","type":"bean","z":"21896b90.de7694","name":"SoilBean","uuid":"","connectiontype":"constant","connectiontimeout":"0"},{"id":"641cbe24.9be34","type":"bean","z":"21896b90.de7694","name":"LiteBean","uuid":"","connectiontype":"constant","connectiontimeout":"0"},{"id":"8dc8c6ca.723738","type":"bean serial","z":"21896b90.de7694","name":"Read from Serial (Temp/Light Sensor)","bean":"641cbe24.9be34","newline":"\\n","bin":"false","out":"char","addchar":false,"x":166,"y":142,"wires":[["8a881461.7577e8"]]},{"id":"295100.ffd6af","type":"file","z":"21896b90.de7694","name":"","filename":"TempLightLog.csv","appendNewline":false,"createDir":true,"overwriteFile":"false","x":474.50006103515625,"y":295,"wires":[]},{"id":"8a881461.7577e8","type":"function","z":"21896b90.de7694","name":"Add Date/Timestamp","func":"\nvar now     = new Date(); \nvar year    = now.getFullYear();\nvar month   = now.getMonth()+1; \nvar day     = now.getDate();\nvar hour    = now.getHours();\nvar minute  = now.getMinutes();\nvar second  = now.getSeconds(); \nif(month.toString().length == 1) {\nvar month = '0'+month;\n}\nif(day.toString().length == 1) {\nvar day = '0'+day;\n}   \nif(hour.toString().length == 1) {\nvar hour = '0'+hour;\n}\nif(minute.toString().length == 1) {\nvar minute = '0'+minute;\n}\nif(second.toString().length == 1) {\nvar second = '0'+second;\n}   \nmsg.timestamp = year+'-'+month+'-'+day+', '+hour+':'+minute+':'+second;\n\nmsg.payload = \n\t{\n\tserial:msg.payload,\n\ttimestamp:msg.timestamp\n\t}\n\t\nreturn msg;","outputs":"1","noerr":0,"x":171.5001220703125,"y":207,"wires":[["d871b321.278e5"]]},{"id":"d871b321.278e5","type":"csv","z":"21896b90.de7694","name":"Format CSV","sep":",","hdrin":"","hdrout":false,"multi":"one","ret":"\\n","temp":"serial , timestamp","x":252.05560302734375,"y":295,"wires":[["295100.ffd6af","4fa3b42b.b05c4c"]]},{"id":"4fa3b42b.b05c4c","type":"debug","z":"21896b90.de7694","name":"","active":true,"console":"false","complete":"payload","x":465.5,"y":333,"wires":[]},{"id":"a050df88.5faf2","type":"bean serial","z":"21896b90.de7694","name":"Read from Serial (Moisture Sensor)","bean":"14230480.ebdcfc","newline":"\\n","bin":"false","out":"char","addchar":false,"x":151,"y":406,"wires":[["832eb29.f7cd15"]]},{"id":"6bff0fbd.9400f","type":"file","z":"21896b90.de7694","name":"","filename":"MoistureLog.csv","appendNewline":false,"createDir":true,"overwriteFile":"false","x":484.50006103515625,"y":551,"wires":[]},{"id":"832eb29.f7cd15","type":"function","z":"21896b90.de7694","name":"Add Date/Timestamp","func":"\nvar now     = new Date(); \nvar year    = now.getFullYear();\nvar month   = now.getMonth()+1; \nvar day     = now.getDate();\nvar hour    = now.getHours();\nvar minute  = now.getMinutes();\nvar second  = now.getSeconds(); \nif(month.toString().length == 1) {\nvar month = '0'+month;\n}\nif(day.toString().length == 1) {\nvar day = '0'+day;\n}   \nif(hour.toString().length == 1) {\nvar hour = '0'+hour;\n}\nif(minute.toString().length == 1) {\nvar minute = '0'+minute;\n}\nif(second.toString().length == 1) {\nvar second = '0'+second;\n}   \nmsg.timestamp = year+'-'+month+'-'+day+', '+hour+':'+minute+':'+second;\n\nmsg.payload = \n\t{\n\tserial:msg.payload,\n\ttimestamp:msg.timestamp\n\t}\n\t\nreturn msg;","outputs":"1","noerr":0,"x":181.5001220703125,"y":463,"wires":[["ebd17285.142e9"]]},{"id":"ebd17285.142e9","type":"csv","z":"21896b90.de7694","name":"Format CSV","sep":",","hdrin":"","hdrout":false,"multi":"one","ret":"\\n","temp":"serial , timestamp","x":262.05560302734375,"y":551,"wires":[["6bff0fbd.9400f","5effc8a9.a10038"]]},{"id":"5effc8a9.a10038","type":"debug","z":"21896b90.de7694","name":"","active":true,"console":"false","complete":"payload","x":475.5,"y":589,"wires":[]}]
```


## Setup the RTC
Typically the Pi retrieves the time from the internet. The external RTC is necessary because this Pi will be used in a location without internet. Currently the Pi is outfitted with a Dallas DS3231 I2C RTC chip, but DS1307 commands work and are the standard.

#### Setup I2C
```bash
sudo apt-get install python-smbus i2c-tools
sudo raspi-config
# Select: Advanced Options > I2C > Yes, enable and yes load by default
sudo reboot
```

- Connect the RTC to the pins shown in the picture and see that it is recognized.
TODO: Add picture
```bash
sudo i2cdetect -y 1
```
- Remove the module blacklist entry so it can be loaded on boot
```bash
sudo sed -i 's/blacklist i2c-bcm2708/#blacklist i2c-bcm2708/' /etc/modprobe.d/raspi-blacklist.conf
```

- Load the module now
```bash
sudo modprobe i2c-bcm2708
```

- Notify Linux of the Dallas RTC device
```bash
echo ds1307 0x68 | sudo tee /sys/class/i2c-adapter/i2c-1/new_device
```

- Test whether Linux can see our RTC module.

```bash
sudo hwclock

# Read the time off the RTC
sudo hwclock -r

# Write system time to the RTC
sudo hwclock -w
```

#### Read from the RTC on Startup

`sudo vim /etc/modules`

```bash
# Add the line to end of file:
rtc-ds1307
```

- Next you will need to add the DS1307 device creation at boot

`sudo vim /etc/rc.local`

```bash
# Add the following lines just before exit 0:
echo ds1307 0x68 > /sys/class/i2c-adapter/i2c-1/new_device
sudo hwclock -s
date
```

#### Prevent RTC from Drifting.
To ensure that the NTP-maintained system clock is periodically saved to the hardware RTC to stop it drifting, I have created in /etc/cron.daily a file called hwRTC. This script updates the RTC from the system clock daily.

`sudo vim /etc/cron.daily/hwRTC`

```
#!/bin/sh
/sbin/hwclock --systohc -D --noadjfile --utc > /tmp/hwClock.log 2>&1
[line break]
```

-  Grant the necessary permisisons to run the file from Cron:

`sudo chmod 755 /etc/cron.daily/hwRTC`

```bash
- Check that it worked:
ls -al hwRTC
# should output something like:
# -rwxr-xr-x 1 root root 79 Jun 24 20:17 hwRTC
```



## Setup USB Flash Drive
Determine the name of the drive.
```bash
df -h
```

Format the drive.
```bash
sudo mkdosfs -F 32 -I /dev/sda1
```

Update the fstab table. Be careful, if you mess up here, the system may not reboot.
```bash
sudo vim /etc/fstab

#Add the following line
/dev/sda1       /media/usbXXX   vfat    users,noauto      0       0
```

Create the mountpoint for the USB. This is the location where files will be saved. Choose any name you like; shorter is better.
```bash
sudo mkdir /media/usbXXX

sudo reboot
```

Test that it worked. After unmounting again, the file test.txt should appear on the UBS drive, even on different computers. If not, something went wrong.
```bash
ls -ld /media/usbXXX
touch /media/usbXXX/test.txt
```


To mount or unmount, use
```bash
mount /dev/sda1
umount /dev/sda1
```

To write to the usb, save files at the location /dev/sda1. See cron for use of this feature.


## Setup Cron
Useful tutorial on Cron jobs: http://www.devils-heaven.com/raspberry-pi-cron-jobs/

Cron is pre-installed on Jessie. To set it up, follow the steps below.

**NOTE: ALWAYS add a line break to your Cron Job**


#### Cron jobs
The following two cron jobs take the csv files created by Node-Red each day, and move them to a separate directory with the date appended to their name. At midnight, a usb flash drive is mounted, and the log directory syncs with the flash drive. After, the flash drive is unmounted. This makes it safe to remove the flash drive at any time (except for a brief window at midnight), without pausing the Pi and data collection.

**Rotate Logs**
```bash
sudo vim /etc/cron.d/rotateLogs

# Write the following lines:
59 23 * * * pi sh /home/pi/Scripts/rotateLogs.sh
[LINE BREAK]
```

**Backup to USB**
```bash
sudo vim /etc/cron.d/backupUSB

# Write the following lines:
@midnight root sh /home/pi/Scripts/backupUSB.sh
[LINE BREAK]
```

Now add the following scripts.
```bash
mkdir /home/pi/Scripts
mkdir /home/pi/Logs

cd /home/pi/Scripts/

# Add the files with the content below
vim rotateLogs.sh
vim backupUSB.sh

# make files executable
chmod u+x *.sh
```


#### File: rotateLogs.sh
```bash
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

#### File: backupUSB.sh
```bash
#!/bin/bash
mount /dev/sda1
rsync -a /home/pi/Logs /media/usbXXX
umount /dev/sda1
```


## Useful Commands
### Applications
- Update Applications
`sudo apt-get update && sudo apt-get dist-upgrade && sudo apt-get autoremove`
- Remove Applications
`sudo apt-get remove --purge libreoffice\*`
`sudo apt-get clean && sudo apt-get autoremove`
- List Installed Applications
`sudo dpkg -l | more` OR `sudo dpkg -l | less`
- List by date added
`cat /var/log/dpkg.log | grep " install "`

### File Management
- See the size of a file
`du -h file.txt`

- Make a file executable
`chmod u+x somefile.sh`

- Remove quotes from a file
`sed -i ’s/\”//g’ file.txt`


### Cron
- View Crontab
`crontab -l`

- Check if Cron is running
`service cron status`

### Bash
- Repeat a command in terminal every x seconds
`watch -n 5 "ls -ltr" `
will run the command "ls -ltr" every 5 seconds (or any other value by replacing 5 with what you want)

- Sudo the previous command
`sudo !!`

- Halt a program/process
`<CONTROL>-C`

- Get current time/date
`date`

- ntp won't set the date/time if the current clock is off by more than 1000 seconds.  The following overrides that restriction.
`sudo ntpd -q -g`

- Restart the Raspberry Pi
`sudo reboot`

- Turn off the PI
`sudo halt -p`

### SSH
- Sync folders over ssh
`rsync -avz -e ssh pi@192.168.0.104:Logs /Users/peterdailey/Desktop/Logs`

- Close ssh connection
`exit`










