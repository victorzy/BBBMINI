# Software

How to prepare your BeagleBone to use as BBBmini.

* Debian 8.6 jessie
* GCC 4.9
* Kernel 4.4 PREEMPT RT
* BBBmini devicetree loaded at startup.

## Prepare microSD with your Linux host computer
1. Download Debian image [https://rcn-ee.net/rootfs/bb.org/testing/2016-10-02/console/BBB-blank-debian-8.6-console-armhf-2016-10-02-2gb.img.xz](https://rcn-ee.net/rootfs/bb.org/testing/2016-10-02/console/BBB-blank-debian-8.6-console-armhf-2016-10-02-2gb.img.xz)
2. Decompress image: `unxz BBB-blank-debian-8.6-console-armhf-2016-10-02-2gb.img.xz`
3. Copy image to microSDcard (>= 2GB): `sudo dd bs=4M if=./BBB-blank-debian-8.6-console-armhf-2016-09-21-2gb.img of=/dev/sdX` /dev/sdX should point to your microSD, be careful here!!! Use `lsblk` to figure out, which is your mircroSD.
The process can take 15-30 minutes depending on the speed of your microSD card.
4. `sync` and remove mircroSD 

## Install Debian to your BeagleBone eMMC
1. Plug prepared microSD into BeagleBone
2. While holding down the boot button, apply power to the board. If there is a newer Debian installed, holding down the boot button is not necessary.
3. Wait some minutes until Debian is installed (all four LEDs turned on).
4. Remove power.
5. Remove microSD.
6. Apply power again.
7. Connect to the BeagleBone `ssh debian@beaglebone`
8. Password `temppwd`
9. Update software: `sudo apt update && sudo apt upgrade -y`
10. Install software: `sudo apt install -y cpufrequtils g++ liblttng-ust-dev pkg-config gawk git make device-tree-compiler screen python python-dev python-lxml python-pip`
11. Install Python library: `sudo pip install future`
12. Set link to pkg-config: `sudo ln -s pkg-config /usr/bin/arm-linux-gnueabihf-pkg-config`
13. Update scripts: `cd /opt/scripts && sudo git pull`
14. Expend partition: `sudo /opt/scripts/tools/grow_partition.sh`
15. Install RT Kernel: `sudo /opt/scripts/tools/update_kernel.sh --bone-rt-kernel --lts-4_4`
16. Add BBBmini DTB: `sudo sed -i 's/#dtb=$/dtb=am335x-boneblack-bbbmini.dtb/' /boot/uEnv.txt`
17. Set clock to fixed 1GHz `sudo sed -i 's/GOVERNOR="ondemand"/GOVERNOR="performance"/g' /etc/init.d/cpufrequtils`
18. Reboot system: `sudo reboot`
19. Login again: `ssh debian@beaglebone`
20. Clone overlays: `git clone https://github.com/beagleboard/bb.org-overlays`
21. Build and install overlays: `cd ./bb.org-overlays && ./install.sh`
22. Add ADC DTBO: `sudo sed -i 's/#cape_enable=bone_capemgr.enable_partno=/cape_enable=bone_capemgr.enable_partno=BB-ADC/g' /boot/uEnv.txt`
23. Reboot system: `sudo reboot`
24. Login again: `ssh debian@beaglebone`
25. Clone ArduPilot code: `git clone https://github.com/ArduPilot/ardupilot.git`
26. Change dir: `cd ardupilot/Tools/Linux_HAL_Essentials/pru/rangefinderpru`
27. Install Rangefinder firmware: `sudo make install`
28. Your BeagleBone is now ready to use.

## Compile ArduPilot native on BeagleBone
1. `cd ardupilot`
2. `git checkout Copter-3.4.4` for ArduCopter or `git checkout ArduPlane-3.7.1` for ArduPlane or `git checkout APMrover2-3.1.0` for ArduRover
3. `git submodule update --init --recursive`
4. `./waf configure --board=bbbmini`
5. `./waf` (take about 1h20m)
6. `cp build/bbbmini/bin/* /home/debian/`

## Cross compile ArduPilot (faster) with Ubuntu computer

Get the source code:

1. `git clone https://github.com/diydrones/ardupilot.git`
2. `cd ardupilot`
3. `./Tools/scripts/install-prereqs-ubuntu.sh`
4. `git checkout Copter-3.4.4` for ArduCopter or `git checkout ArduPlane-3.7.1` for ArduPlane or `git checkout APMrover2-3.1.0` for ArduRover
5. `git submodule update --init --recursive`
6. `./waf configure --board=bbbmini`
7. `./waf`
8. `scp build/bbbmini/bin/* debian@beaglebone:/home/debian/`

## Run ArduPilot
Now you can check your hardware [here.](../checkhardware/checkhardware.md)

ArduCopter:
`sudo /home/debian/arducopter-quad` (plus parameter) 

ArduPlane:
`sudo /home/debian/arduplane` (plus parameter) 

ArduRover:
`sudo /home/debian/ardurover` (plus parameter) 

To connect a MAVLink groundstation with IP 192.168.178.26 add `-C udp:192.168.178.26:14550`

To use MAVLink via radio connected to UART4 add `-C /dev/ttyO4`. 

If there is a GPS connected to UART5 add `-B /dev/ttyO5`. 

Example: MAVLink groundstation with IP 192.168.178.26 on port 14550 and GPS connected to `/dev/ttyO5` UART5.

`sudo /home/debian/arducopter-quad -C udp:192.168.178.26:14550 -B /dev/ttyO5`

Example: MAVLink groundstation via radio connected to UART4 and GPS connected to `/dev/ttyO5` UART5.

`sudo /home/debian/arducopter-quad -B /dev/ttyO5 -C /dev/ttyO4`

## Automatic start ArduPilot after boot

If ArduPilot should start automatically at boot time follow the instructions below:

1. Connect to your BeagleBone via ssh with `ssh debian@beaglebone`
2. Edit `/etc/rc.local` with `sudo nano /etc/rc.local`
3. Modify file to (use your ArduPilot file and parameter):
```
#!/bin/sh -e
#
# rc.local
#
# This script is executed at the end of each multiuser runlevel.
# Make sure that the script will "exit 0" on success or any other
# value on error.
#
# In order to enable or disable this script just change the execution
# bits.
#
# By default this script does nothing.

/bin/sleep 10
/home/debian/arducopter-quad -B /dev/ttyO5 -C /dev/ttyO4 > /home/debian/arducopter.log &

exit 0
```
4. Save file: `Strg + o` + Enter
5. Exit nano: `Strg + x`
6. Reboot BegaleBone with `sudo reboot`
