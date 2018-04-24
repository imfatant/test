# ArduPilot Blue - a beginner's guide.
This is my continuously updated (as of 24/04/2018, UK date format) beginner's guide for setting up the BeagleBone Blue with Mirko Denecke's port of ArduPilot (https://github.com/mirkix/ardupilotblue). This guide is based in great part upon a similar one at Patrick Poirier's PocketPilot project (https://github.com/PocketPilot/PocketPilot). Many thanks to both these guys, and others too, for their superb work!

Only necessary steps are shown, excepting that I install Git for the sake of convenience. Information found elsewhere may indicate steps that are no longer necessary due to the software having been updated, or steps that only apply to other platforms (BBBMINI or PocketPilot, etc).

# Part 1.
0) Before I begin, I want to stress that supplying adequate power to the BeageBone Blue is a must. Typically, we'll be attaching quite a few peripherals to it, and they'll not behave correctly without enough juice.

1) Go to https://rcn-ee.net/rootfs/bb.org/testing/ and select the directory named with the latest date. Then click on the stretch-console directory. You'll see a number of files here. Download the file named something like bone-debian-V.V-console-armhf-20YY-MM-DD-1gb.img.xz. This is what's known as the 'console' image. It's a very minimal image of Debian with only the bare essentials. An alternative is the 'IoT' image (IoT = Internet of Things) which comes with additional software and can make for a more comfortable experience if you are very new to Linux. It's available from the same site.

    At the time of writing, I'm using bone-debian-9.4-console-armhf-2018-04-22-1gb.img.xz.

2) Now you'll need to copy the image to a microSD card. Whether you are using Linux or Windows, I highly recommend a program called Etcher for this task (https://etcher.io/).

3) It should now be possible to boot up the BeagleBone Blue from the microSD card. It is beyond the scope of this document to detail ways of interacting with the BBBlue, but often it's accomplished by plugging in a Micro-USB cable and either using SSH (to 'debian@192.168.7.2', password 'temppwd') or establishing a serial link over a COM port (user 'debian', password 'temppwd)'. More information can be found here: https://beagleboard.org/blue.

4) Hopefully, you now find yourself logged into the debian user account and at the command prompt. The next task is to update and install some software using an available internet connection.

    If you are SSHing to 192.168.7.2, you can your share your computer's internet connection with the BBBlue very simply by typing (at the BBBlue's command prompt): `sudo /sbin/route add default gw 192.168.7.1`
    
    If you have established a serial link (in a terminal program like Minicom), the process is more difficult, especially as the console image does not include Connman (for WiFi). In the end, I chose to use a USB-to-Ethernet dongle I had lying around (http://accessories.ap.dell.com/sna/productdetail.aspx?c=sg&l=en&s=bsd&cs=sgbsd1&sku=470-ABNL) as I could plug this into the BBBlue and then connect it directly to my router. Note that I had to supply extra power to the BBBlue via its 2s LiPo connector for the dongle to work. The BBBlue enumerates the dongle as device 'usb2', and sets it up automatically (well, sometimes it's necessary to reboot in order for the command `ip link` to show that the device is up).
    
    Whichever way you go, you can verify you have internet by typing: `ping -c 3 www.google.com`

5) Update and install all required supporting software:

        sudo apt-get -y update
        sudo apt-get -y dist-upgrade
        sudo apt-get install -y cpufrequtils connman git
6) Update Git: `cd /opt/scripts && git pull`
7) Specify Ti real-time kernel 4_9. Do NOT use 4_14: `sudo /opt/scripts/tools/update_kernel.sh --ti-rt-channel --lts-4_9`
8) Specify device tree binary to be used at startup: `sudo sed -i 's/#dtb=/dtb=am335x-boneblue.dtb/g' /boot/uEnv.txt`
9) Set clock frequency: `sudo sed -i 's/GOVERNOR="ondemand"/GOVERNOR="performance"/g' /etc/init.d/cpufrequtils`
10) Disable Bluetooth (optional): `sudo systemctl disable bb-wl18xx-bluetooth.service`
11) Maximize the microSD card's existing partition (which is /dev/mmcblk0p1): `sudo /opt/scripts/tools/grow_partition.sh`
12) Reboot now: `sudo reboot`
13) It's time to set up Connman for WiFi. I do it the following way because it's easier to automate in a script later on. First, make a note of your router's SSID and WiFi password. Then type the following:

        su
        (enter password 'root')
        connmanctl services | grep '<your SSID>' | grep -Po 'wifi_[^ ]+'
    The response will be a hash that'll look something like 'wifi_38d279e099a8_4254487562142d4355434b_managed_psk'. If you see nothing, try it again - you probably made a typo.
    
    Now, using this hash, we're going to enter a file directly from the keyboard (stdin) using cat, one line at a time:
    
        cat >/var/lib/connman/wifi.config
        [service_<your hash>]
        Type = wifi
        Security = wpa2
        Name = <your SSID>
        Passphrase = <your WiFi password>
        (press Ctrl-D to exit back to the prompt)
    Finally, type:

        systemctl enable connman
14) Reboot again now: `reboot`

# Part 2.
15) When the BBBlue comes back up, you should notice that a prominent green LED is lit, signifying that WiFi is working. And by the way, if you used a USB-to-Ethernet dongle in Step 4, unplug it now, and hopefully your green LED will go on, too.

16) Now we need to create a few text files. Use your favourite text editor (with sudo). Personally, I like nano. First, the ArduPilot environment configuration file, /etc/default/ardupilot:
        
        TELEM1="-C /dev/ttyO1"
        TELEM2="-A udp:192.168.0.13:14550"
        GPS="-B /dev/ttyS2"
    This is a pretty typical config. It means the following:
    
    Switch -C links ArduPilot's Telem1 serial port (SERIAL1) with the BBBlue's UART1. For example, I have a RFDesign 868x radio modem connected to UART1. It is the bidirectional datalink with my drone. It sends various telemetry data to the base station, and receives commands and RTK differential corrections from the base station.
    
    Switch -A links ArduPilot's Console serial port (SERIAL0) with a protocol, IP address and port number of one's choosing. For example, this allows me to have MAVLink data coming over WiFi for test purposes. Really useful, since it seems to be reliably auto-sensed by ground control station software like Mission Planner and QGroundControl.
    
    Switch -B links ArduPilot's GPS serial port (SERIAL3) with the BBBlue's UART2 (the UART named 'GPS' on the board itself). For example, I have a u-blox NEO-M8P connected to UART2.
    
    Other possibilities exist, namely:
    
        Switch -A ---> SERIAL0
        Switch -B ---> SERIAL3
        Switch -C ---> SERIAL1
        Switch -D ---> SERIAL2
        Switch -E ---> SERIAL4
        Switch -F ---> SERIAL5
17) Next, we'll create the ArduPilot systemd service files, one for ArduPlane, /lib/systemd/system/arduplane.service:
    
        [Unit]
        Description=ArduPlane Service
        After=networking.service
        StartLimitIntervalSec=0
        Conflicts=arducopter.service ardupilot.service ardurover.service

        [Service]
        EnvironmentFile=/etc/default/ardupilot
        ExecStartPre=/usr/bin/ardupilot/ap
        ExecStart=/usr/bin/ardupilot/arduplane $TELEM1 $TELEM2 $GPS

        Restart=on-failure
        RestartSec=1

        [Install]
        WantedBy=multi-user.target
    And one for ArduCopter, /lib/systemd/system/arducopter.service:
    
        [Unit]
        Description=ArduCopter Service
        After=networking.service
        StartLimitIntervalSec=0
        Conflicts=arduplane.service ardupilot.service ardurover.service

        [Service]
        EnvironmentFile=/etc/default/ardupilot
        ExecStartPre=/usr/bin/ardupilot/ap
        ExecStart=/usr/bin/ardupilot/arducopter $TELEM1 $TELEM2 $GPS

        Restart=on-failure
        RestartSec=1

        [Install]
        WantedBy=multi-user.target
18) Pause for a moment to create a directory: `sudo mkdir -p /usr/bin/ardupilot`
    
    Then carry on with what I call the ArduPilot hardware configuration file, /usr/bin/ardupilot/ap, which is run by the services prior to running the ArduPlane or ArduCopter executables:

        #!/bin/bash
        # ap
        # ArduPilot hardware config.

        /usr/bin/echo 80 >/sys/class/gpio/export
        /usr/bin/echo out >/sys/class/gpio/gpio80/direction
        /usr/bin/echo 1 >/sys/class/gpio/gpio80/value
        /usr/bin/echo pruecapin_pu >/sys/devices/platform/ocp/ocp:P8_15_pinmux/state
    You may want to use `sudo chmod 0744 /usr/bin/ardupilot/ap` to set permissions for this file.
    
19) Almost there! You must now obtain the latest ArduPlane and ArduCopter executables and place them in the /usr/bin/ardupilot directory. Depending on the situation, this may mean building them from scratch. Compiling them on the BBBlue itself is an option, but takes an absolute age. Patrick explains the process for the BBBMINI (based on a BeagleBone Black) here: https://github.com/mirkix/BBBMINI/blob/master/doc/software/software.md. For now, I will run through the process of cross-compiling them on a relatively powerful desktop PC running Arch Linux (using pacaur rather than yaourt), which is a lot faster:

        gpg --recv-keys 79BE3E4300411886 38DBBDC86092693E 79C43DFBF1CF2187
        pacaur -S arm-linux-gnueabihf-gcc  # <--- Note that I'm using pacaur instead of yaourt. Also, ensure you haven't set any C/C++ env variables.
        sudo ln -s pkg-config /usr/bin/arm-linux-gnueabihf-pkg-config
        sudo pip install future
        git clone https://github.com/diydrones/ardupilot.git
        cd ardupilot
        git config user.name <your username>
        sed -i 's/command -v yaourt/command -v pacaur/g' ./Tools/scripts/install-prereqs-arch.sh  # <--- Skip this if using yaourt.
        sed -i 's/yaourt -S --noconfirm --needed/pacaur -S --noconfirm --noedit/g' ./Tools/scripts/install-prereqs-arch.sh  # <--- Skip if using yaourt.
        git commit -a --allow-empty-message -m ''  # <--- The lazy option.
        ./Tools/scripts/install-prereqs-arch.sh
        git checkout Copter-3.5.5  # <--- For ArduCopter. For ArduPlane, use: git checkout ArduPlane-3.8.4
        git submodule update --init --recursive
        ./waf configure --board=blue  # <--- BeagleBone Blue.
        ./waf
         scp ./build/blue/bin/ardu* debian@192.168.7.2:/home/debian  # <--- Finally, copy the built executable(s) over to the BBBlue.
    Log in to the BBBlue and copy the executable(s) from /home/debian to /usr/bin/ardupilot with: `sudo cp /home/debian/ardu* /usr/bin/ardupilot`
    
20) To get ArduPilot going, choose which flavour you want and type either:

        sudo systemctl enable arduplane.service
    Or:

        sudo systemctl enable arducopter.service
    After you reboot, your ArduPilot should inflate automatically.
    
-- Imf
