# ArduPilot Blue - A beginner's guide
This is my regularly updated (as of 20/05/2018, DD/MM/20YY) beginner's guide to setting up the BeagleBone Blue with Mirko Denecke's port of ArduPilot (https://github.com/mirkix/ardupilotblue). It is based in great part upon similar documents at Patrick Poirier's PocketPilot project (https://github.com/PocketPilot/PocketPilot). Many thanks to both these fine individuals, and others too @https://gitter.im/mirkix/BBBMINI, for their superb work!

I take a minimalistic approach. Only necessary steps are shown, excepting that I install Git for the sake of convenience. Information found elsewhere may indicate steps that are no longer necessary due to software updates, or steps that only apply to other platforms (BBBMINI or PocketPilot, etc).

## Part 1 - Preparation
0) Before I begin, I want to stress that supplying the BeagleBone Blue with adequate power is a must. Typically, we'll be attaching quite a few navigation-related peripherals to it, and they'll not behave correctly without enough juice.

1) Go to https://rcn-ee.net/rootfs/bb.org/testing/ and select the directory named with the latest date. Then click on the stretch-console subdirectory. You'll see a number of files here. Download the file named something like 'bone-debian-V.V-console-armhf-20YY-MM-DD-1gb.img.xz'. This is what's known as the 'console' image. It's a very minimal distribution of Debian with only the bare essentials. An alternative is the 'IoT' image (IoT = Internet of Things) which comes with additional software and can make for a more comfortable experience if you are very new to Linux. It's available from the same site.

    Don't assume that the latest image is the best, or even that it works. Afterall, this is the 'testing' repository, so be warned. However, I will endeavour to keep abreast of what seems to be the latest functional image and mention it here.

    At the time of writing, I'm using https://rcn-ee.net/rootfs/bb.org/testing/2018-04-22/stretch-console/bone-debian-9.4-console-armhf-2018-04-22-1gb.img.xz.
    
    PLEASE TRY USING THIS PRECISE IMAGE FIRST BEFORE RAISING ISSUES!
    
2) Next you'll need to flash the image to a microSD card. Whether you are using Linux or Windows, I highly recommend a program called Etcher for this task (https://etcher.io/).

3) It should now be possible to boot up the BeagleBone Blue from the microSD card. It's beyond the scope of this document to detail all the ways of interacting with the BBBlue, but often it's accomplished by plugging in a Micro-USB cable and either using SSH (to 'debian@192.168.7.2', password 'temppwd') or establishing a serial link over a COM port (user 'debian', password 'temppwd)' in a program like Minicom or PuTTY. More information can be found here: https://beagleboard.org/blue

    BeagleBone drivers come with Windows 10, but Linux distributions can be hit-and-miss. If you're experiencing problems, try making a udev rule:

       sudo -s
       cat >/etc/udev/rules.d/73-beaglebone.rules <<EOF
       ACTION=="add", SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_interface", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="a6d0", DRIVER=="", RUN+="/sbin/modprobe -b ftdi_sio"
       ACTION=="add", SUBSYSTEM=="drivers", ENV{DEVPATH}=="/bus/usb-serial/drivers/ftdi_sio", ATTR{new_id}="0403 a6d0"
       ACTION=="add", KERNEL=="ttyUSB*", ATTRS{interface}=="BeagleBone", ATTRS{bInterfaceNumber}=="00", SYMLINK+="beaglebone-jtag"
       ACTION=="add", KERNEL=="ttyUSB*", ATTRS{interface}=="BeagleBone", ATTRS{bInterfaceNumber}=="01", SYMLINK+="beaglebone-serial"
       EOF
       udevadm control --reload-rules
       exit

4) Hopefully, you now find yourself logged in to the BBBlue and at the command prompt. The next job is to update and install some software using an available internet connection, so it's time to set up connman for WiFi access. I do it the following way because it's easier to automate in a script later on. First, make a note of your router's SSID and WiFi password. Then type the following:

        sudo -s
        connmanctl services | grep '<your SSID>' | grep -Po 'wifi_[^ ]+'
    The response will be a hash that'll look something like 'wifi_38d279e099a8_4254487562142d4355434b_managed_psk'. If you see nothing, try it again - you probably made a typo.
    
    Now, using this hash, we're going to enter a file directly from the keyboard (stdin) using `cat`, one line at a time:
    
        cat >/var/lib/connman/wifi.config
        [service_<your hash>]
        Type = wifi
        Security = wpa2
        Name = <your SSID>
        Passphrase = <your WiFi password>
    Press `Ctrl-D` to quit out to the prompt, and then type: `exit`
    
    Wait a second and a prominent green LED will come on, signifying that WiFi is up. The BBBlue is connected to your router, and its address on your WiFi network can be found using programs like nmap: `sudo nmap 192.168.0.0/24`. It's also possible that you can find the address by logging in to your router and looking there. Try SSHing to the BBBlue using its WiFi IP address.
    
    192.168.7.2 will still work as well.
    
    If you can't get WiFi to work with connman, or you simply don't want to use connman, you can use the following method. First, type: `sudo systemctl disable connman`. Then edit /etc/network/interfaces to read:

       # The loopback network interface.
       auto lo
       iface lo inet loopback

       # WiFi w/ onboard device (dynamic IP).
       auto wlan0
       iface wlan0 inet dhcp
       wpa-ssid "<your SSID>"
       wpa-psk "<your WiFi password>"
       
       # Ethernet/RNDIS gadget (g_ether).
       # Used by: /opt/scripts/boot/autoconfigure_usb0.sh
       iface usb0 inet static
       address 192.168.7.2
       netmask 255.255.255.252
       network 192.168.7.0
       gateway 192.168.7.1
    Finally, reboot the BBBlue with: `sudo reboot`
    
    After powering back on, type:
    
       sudo ifup wlan0
       echo "nameserver 8.8.8.8" | sudo tee -a /etc/resolv.conf >/dev/null
    The green LED should come on. You're connected.
    
    If you want the BBBlue to have a static IP (let's say 192.168.0.99), alter the relevant section of /etc/network/interfaces to read:
    
       # WiFi w/ onboard device (static IP).
       auto wlan0
       iface wlan0 inet static
       wpa-ssid "<your SSID>"
       wpa-psk "<your WiFi password>"
       address 192.168.0.99  # <--- The desired static IP address of the BBBlue.
       netmask 255.255.255.0
       gateway 192.168.0.1  # <--- The address of your router.
    If WiFi just isn't an option, you can tell the BBBlue that it'll be sharing your computer's internet connection by typing (at the BBBlue's command prompt):
    
       sudo /sbin/route add default gw 192.168.7.1
       echo "nameserver 8.8.8.8" | sudo tee -a /etc/resolv.conf >/dev/null
    You can then tell your Linux computer to share with the BBBlue by typing (at the computer's command prompt):
    
       sudo sysctl net.ipv4.ip_forward=1
       sudo iptables -t nat -A POSTROUTING -o eno1 -j MASQUERADE
       sudo iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
       sudo iptables -A FORWARD -i enp0s20u7 -o eno1 -j ACCEPT
    On my Arch Linux desktop x64 PC, 'eno1' is the name of the Ethernet adapter connected to my router (and the connection I'll be sharing), and 'enp0s20u7' is the name assigned to the BBBlue. Use `ip link` to find the corresponding names for your devices.
    
    If you are using a Windows computer, you can tell it to share with the BBBlue by fast-forwarding to about 5:00 of this Derek Molloy video: http://derekmolloy.ie/beaglebone/getting-started-usb-network-adapter-on-the-beaglebone/
    
    If internet sharing isn't an option, another possibility is a USB-to-Ethernet dongle like this one: http://accessories.ap.dell.com/sna/productdetail.aspx?c=sg&l=en&s=bsd&cs=sgbsd1&sku=470-ABNL). One can plug this into the BBBlue and then connect it directly to the router. Note that, in my case, I had to supply extra power to the BBBlue via its 2s LiPo connector or jack plug for the dongle to work. The BBBlue enumerates the dongle as device 'usb2', and sets it up automatically (well, sometimes it's necessary to reboot in order for `ip link` to show that the device is up).
    
    Whichever way you go, you can verify you have internet by typing: `ping -c 3 google.com`

5) Update and install all required supporting software:

       sudo apt-get -y update
       sudo apt-get -y dist-upgrade
       sudo apt-get install -y cpufrequtils git
6) Update scripts: `cd /opt/scripts && git pull`
7) Specify Ti real-time kernel 4_9. Do NOT use 4_14: `sudo /opt/scripts/tools/update_kernel.sh --ti-rt-channel --lts-4_9`
8) Specify device tree binary to be used at startup: `sudo sed -i 's/#dtb=/dtb=am335x-boneblue.dtb/g' /boot/uEnv.txt`
9) Set clock frequency: `sudo sed -i 's/GOVERNOR="ondemand"/GOVERNOR="performance"/g' /etc/init.d/cpufrequtils`
10) Disable Bluetooth (optional): `sudo systemctl disable bb-wl18xx-bluetooth.service`
11) Maximize the microSD card's existing partition (which is /dev/mmcblk0p1): `sudo /opt/scripts/tools/grow_partition.sh`
12) Reboot now: `sudo reboot`

## Part 2 - Putting ArduPilot on the BeagleBone Blue

13) When the BBBlue comes back up, we need to create a few text files. Use your favourite text editor. Personally, I like nano, which, owing to the way these Debian images have been configured, is invoked by default if you use the `sudoedit` command. First, the ArduPilot environment configuration file, /etc/default/ardupilot:
        
        TELEM1="-C /dev/ttyO1"
        TELEM2="-A udp:192.168.0.13:14550"
        GPS="-B /dev/ttyS2"
    This is a pretty typical config. It breaks down like this:
    
    Switch -C maps ArduPilot's "Telem1" serial port (SERIAL1, default 57600) to the BBBlue's UART1. For example, I have a RFDesign 868x radio modem connected to UART1. It is the bidirectional data link with my drone. It sends various telemetry data to the base station, and receives commands and RTK differential corrections from the base station.
    
    Switch -A maps ArduPilot's "Console" serial port (SERIAL0, default 115200) to a protocol, IP address and port number of one's choosing. For example, this allows me to have MAVLink data coming over WiFi for test purposes. Really useful, especially since it seems to be reliably auto-sensed by ground control station software like Mission Planner and QGroundControl.
    
    Switch -B maps ArduPilot's "GPS" serial port (SERIAL3, default 57600) to the BBBlue's UART2 (the UART named 'GPS' on the board itself). For example, I have a u-blox NEO-M8P connected to UART2.
    
    Other possibilities exist, namely:
    
        Switch -A ---> "Console", SERIAL0, default 115200
        Switch -B ---> "GPS", SERIAL3, default 57600
        Switch -C ---> "Telem1", SERIAL1, default 57600
        Switch -D ---> "Telem2", SERIAL2, default 38400
        Switch -E ---> Unnamed, SERIAL4, default 38400
        Switch -F ---> Unnamed, SERIAL5, default 57600
    Consult the official ArduPilot documentation for more details on the various serial ports: http://ardupilot.org/plane/docs/parameters.html?highlight=parameters
    
14) Next, we'll create the ArduPilot systemd service files, one for ArduCopter, /lib/systemd/system/arducopter.service:
        
        [Unit]
        Description=ArduCopter Service
        After=networking.service
        StartLimitIntervalSec=0
        Conflicts=arduplane.service ardurover.service antennatracker.service

        [Service]
        EnvironmentFile=/etc/default/ardupilot
        ExecStartPre=/usr/bin/ardupilot/aphw
        ExecStart=/usr/bin/ardupilot/arducopter $TELEM1 $TELEM2 $GPS

        Restart=on-failure
        RestartSec=1

        [Install]
        WantedBy=multi-user.target
    One for ArduPlane, /lib/systemd/system/arduplane.service:
        
        [Unit]
        Description=ArduPlane Service
        After=networking.service
        StartLimitIntervalSec=0
        Conflicts=arducopter.service ardurover.service antennatracker.service

        [Service]
        EnvironmentFile=/etc/default/ardupilot
        ExecStartPre=/usr/bin/ardupilot/aphw
        ExecStart=/usr/bin/ardupilot/arduplane $TELEM1 $TELEM2 $GPS

        Restart=on-failure
        RestartSec=1

        [Install]
        WantedBy=multi-user.target    
    And one for ArduRover, /lib/systemd/system/ardurover.service:
    
        [Unit]
        Description=ArduRover Service
        After=networking.service
        StartLimitIntervalSec=0
        Conflicts=arducopter.service arduplane.service antennatracker.service

        [Service]
        EnvironmentFile=/etc/default/ardupilot
        ExecStartPre=/usr/bin/ardupilot/aphw
        ExecStart=/usr/bin/ardupilot/ardurover $TELEM1 $TELEM2 $GPS

        Restart=on-failure
        RestartSec=1

        [Install]
        WantedBy=multi-user.target
    While I'm here, how about AntennaTracker, too? Create /lib/systemd/system/antennatracker.service:
    
        [Unit]
        Description=AntennaTracker Service
        After=networking.service
        StartLimitIntervalSec=0
        Conflicts=arducopter.service arduplane.service ardurover.service

        [Service]
        EnvironmentFile=/etc/default/ardupilot
        ExecStartPre=/usr/bin/ardupilot/aphw
        ExecStart=/usr/bin/ardupilot/antennatracker $TELEM1 $TELEM2 $GPS

        Restart=on-failure
        RestartSec=1

        [Install]
        WantedBy=multi-user.target
15) Pause for a moment to create a directory: `sudo mkdir -p /usr/bin/ardupilot`
    
    Then, carry on with creating what I call the ArduPilot hardware configuration file, /usr/bin/ardupilot/aphw, which is run by the services prior to running the ArduPilot executables:

        #!/bin/bash
        # aphw
        # ArduPilot hardware configuration.

        /bin/echo 80 >/sys/class/gpio/export
        /bin/echo out >/sys/class/gpio/gpio80/direction
        /bin/echo 1 >/sys/class/gpio/gpio80/value
        /bin/echo pruecapin_pu >/sys/devices/platform/ocp/ocp:P8_15_pinmux/state
    Use `sudo chmod 0755 /usr/bin/ardupilot/aphw` to set permissions for this file.
    
16) Almost there! You must now obtain the latest ArduCopter, ArduPlane, etc. executables, built specifically for the BBBlue's Arm architecture, and place them in the /usr/bin/ardupilot directory. Mirko Denecke has them on his site here: http://bbbmini.org/download/blue/

    And I've built my own copies here in this repository: https://github.com/imfatant/test/blob/master/bin/

    Be sure to set their permissions with: `sudo chmod 0755 /usr/bin/ardupilot/a*`

    If you find that you need to build them from scratch yourself, do not be intimidated - this is not too difficult. Plus it means you'll be able to build your own customized ArduPilot software.

    Compiling them on the BBBlue itself is an option, but takes an absolute age. Patrick Poirier explains the process for the BBBMINI (based on a BeagleBone Black) here: https://github.com/mirkix/BBBMINI/blob/master/doc/software/software.md. Fortunately, he also provides instructions to cross-compile them on a relatively powerful desktop x64 PC in Ubuntu, which is much, much faster. Here, I will run through the process of cross-compiling them in Arch Linux (which happens to be God's Own Linux Distro):

        sudo pacman -Syu
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
        git fetch --prune  # <--- Updates the repository.
        git checkout Copter-3.5.5  # <--- For ArduCopter. For ArduPlane, use: git checkout ArduPlane-3.8.5
        git submodule update --init --recursive
        ./waf configure --board=blue  # <--- BeagleBone Blue.
        ./waf
         scp ./build/blue/bin/ardu* debian@192.168.7.2:/home/debian  # <--- Finally, copy the built executable(s) over to the BBBlue.
    Log in to the BBBlue and copy the executable(s) from /home/debian to /usr/bin/ardupilot with: `sudo cp /home/debian/ardu* /usr/bin/ardupilot`
    
    Again, be sure to set their permissions: `sudo chmod 0755 /usr/bin/ardupilot/a*`
    
17) To get ArduPilot going, choose which flavour you want and type:

        sudo systemctl enable arducopter.service
    Or:

        sudo systemctl enable arduplane.service
    Or:

        sudo systemctl enable ardurover.service
	
	Or:
	
        sudo systemctl enable antennatracker.service
    After you reboot, your ArduPilot should inflate automatically.

## Part 3 - Connecting the peripherals
18) A basic minimum configuration is likely to include:
    - An R/C receiver.
    - A GPS receiver (with or without integrated compass).
    - A radio modem for a bidirectional data link, particularly at longer ranges.

    (The BBBlue's onboard WiFi is great for debugging and testing at close range if 2.4 GHz is available, but for anything more interesting, a dedicated bidirectional data link is recommended. Also bear in mind the type and placement of antennas that all these items use.)

    A good place to begin is the quickstart pinout diagram below (save the file and open it in any good image viewer for better resolution). I will refer to it in the paragraphs that follow.
    
    But first a few words about connectors, cables, and the tools you'll need, particularly if you're to make up your own cables. I'm going to make a few strong recommendations because if you're new to this, you could waste a great deal of time, effort and money mucking about. The connector type mainly used is the JST-SH 1.0 mm pitch. You should buy some females, in 4 and 6-way sizes, and the crimp contacts to go with (buy lots so you can practise - you'll need to). Then, get a couple of metres of 28 or 30 AWG silicone covered wire, in at least five different colours (red, black, yellow, green, blue and white), Engineer PA-06 wire strippers with the depth gauge attachment, and Engineer PA-09 'connector pliers' (they're Japanese - we tend to call them crimpers). By the way, the JST-SH is a horrible connector. It may be tiny, but that makes it hard to work with and it has a habit of working loose.
    
    I should also mention other connector types you'll come across (on the other end of the cables you'll be making up). The JST-GH 1.25 mm pitch is quite common, and I hope that it will be more widely adopted. It's my favourite connector - sturdy and has an integrated catch mechanism. In the wild, you'll see 4, 5 and 6-way. Next, there's the Molex PicoBlade 1.25 mm pitch connector, also nice, and available in 'free floating' female (called a 'receptacle') and male (called a 'plug housing') varieties - great for field-pluggable connections. Grab some from 2-way through to 6-way. Finally, there's the venerable Mini-PV 2.54 mm connector (a.k.a. Harwin M20 / Futaba / Hitec / 'standard black R/C-style connector'). Naturally, you'll need to buy the crimp contacts for all these (they're all different).
    
    DigiKey, Mouser, RS Components, Farnell, and so on, are the places to shop for these components. Try to make one BIG order because they charge a small fortune for shipping. As for information on all the rest of the bits and pieces you'll need, go here to Joshua Bardwell's FPV website: https://www.fpvknowitall.com/ultimate-fpv-shopping-list/
    
    So, back to connecting the peripherals:
    
![alt text](https://github.com/imfatant/test/blob/master/docs/BBBlue-ArduPilot.jpg)

- The R/C receiver: FrSky equipment (https://www.frsky-rc.com/), like the R-XSR, XR4SB, X6R, X8R or R9 Slim can be powered off any +5V pin and a GND. The +5V servo rail, providing it's powered, will of course do. All that remains is to connect the receiver's SBUS OUT to one of the two SBUS pins marked on the diagram. A signal inverter is not necessary. Please contact me if you would like to add some information here about gear from other manufacturers such as Spektrum, Futaba, etc.

- The GPS receiver: Most people use the u-blox receivers, particularly the NEO-M8N and NEO-M8P. The former can be easily and cheaply obtained online from Chinese companies like HobbyKing (https://hobbyking.com/en_us/catalogsearch/result/?cat=&q=ublox), in a variety of usually disc-shaped packages. Conveniently, these packages contain the receiver itself, a very small ceramic patch antenna, and often include a solid-state compass. While the BBBlue already has an onboard compass (the AKM AK8963), ArduPilot can be configured to use this 'external' compass instead, something to consider if it's positioned in a less (electro)magnetically noisy location on the drone.

    The (much) more expensive alternative to the NEO-M8N is the NEO-M8P. This receiver supports a mode of operation known as 'RTK', or Real-Time Kinematic. What that means is that we can achieve positional accuracies of a few centimetres in real-time. At the time of writing, this kind of performance comes at a cost, approximately ten times the cost of the NEO-M8N, and that's before you add in the base station. I will append a special section addressing the M8P later in the guide.
    
    I highly recommend starting out with an M8N, if only just to get a feel for things. It's highly likely you'll have to chop off the connector(s) it comes with and break out the GPS lines to a 6-way JST-SH which you'll plug into the GPS serial UART on the BBBlue, and the external compass lines to a 4-way JST-SH which plugs into the I2C port. The niggle is that although you'll be powering the M8N with +5V, its GPS signal must not exceed +3.3V. I believe most M8N modules step the voltage down to +3.3V internally, so you're OK. But you should check, if necessary with an oscilloscope. Failure to do so could damage the BBBlue.
    
    ... under construction ... 
    
The ArduPilot parameter settings files: /var/APM/{ArduCopter.stg,ArduPlane.stg,APMrover2.stg,AntennaTracker.stg}
...

sudo apt-get install i2c-tools

sudo i2cdetect -r -y 0
sudo i2cdetect -r -y 1
sudo i2cdetect -r -y 2

    $ sudo i2cdetect -r -y 2
    0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
    00:          -- -- -- -- -- -- -- -- -- 0c -- -- -- 
    10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
    20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
    30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
    40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
    50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
    60: -- -- -- -- -- -- -- -- 68 -- -- -- -- -- -- -- 
    70: -- -- -- -- -- -- 76 --    
68 = InvenSense MPU-9250 IMU (onboard), 0c = AKM AK8963 compass (onboard), 76 = Bosch BMP280 barometer (onboard).

    $ sudo i2cdetect -r -y 1
    0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
    00:          -- -- -- -- -- -- -- -- -- -- -- -- -- 
    10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- 1e -- 
    20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
    30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
    40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
    50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
    60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
    70: -- -- -- -- -- -- -- --
1e = Honeywell HMC5843 compass (external) - often comes integrated into the inexpensive u-blox NEO-M8N-based GPS modules.



	#!/bin/bash
	# bbblue49-ardupilot-setup
	# We are user..
	# Sets up ArduPilot on a BeagleBone Blue remote drone server over SSH, using 'console' or 'IoT' Debian image as a base.
	# Additionally, sets up a squawk, and flashes everything to eMMC should you leave the relevent lines uncommented.
	# NB: You may need to configure your router's port-forwarding to handle SSH traffic on port 22.
	# NB: DO NOT ALTER THE FORMATTING OF THIS SCRIPT. IT USES TAB NESTED HERE DOCUMENTS!
	# NB: SOME LINES ARE INTENTIONALLY LEFT BLANK!
	
	# Before running this script, you must do the following on the local machine:
	# 1. Have your SSH service ready & waiting for the remote server to squawk back at you - typically with `systemctl enable --now sshd.service`.
	# This is important because it will mean that /etc/ssh/ssh_host_ecdsa_key.pub actually exists, which is used in the script (alternatively, ssh-keygen -A).
	# 2. Run `ssh-keygen -b 4096` if you haven't yet created your personal SSH private/public key pair.
	# 3. Have the following files in your ~ directory: .bash_profile, .bashrc, arducopter, arduplane, ardurover and antennatracker. DIR_COLORS must be in /etc.
	# 4. Disable any VPN because $local-ip may be assigned the 'wrong' address.
	# 5. If it already exists, delete the BBBlue's entry in ~/.ssh/known_hosts. Otherwise, script will fail with warnings of possible attack, etc.
	# 6. If it exists, delete relevant entry in ~/.ssh/authorized_keys.
	# 7. Consider whether the remote server needs a geo-tailored mirror list.
	
	# Comments:
	# The Debian images have root login disabled, but the default user (debian) has sudo privs.
	# As it stands, I fetch the remote server's pub keys at the end.
	# I believe SSH key permissions are set correctly.
	# Debian enumerates the BBBlue's standard network link options as wlan0, usb0 and usb1.
	
	# User configuration starts here:
	local_user="<your username>"  # <--- SET THIS.
	local_ip="$(ip route get 8.8.8.8 | awk '{print $7; exit}')"  # Private or public IP as appropriate.
	local_port="14550"  # <--- SET THIS (for ArduPilot).
	local_protocol="udp"  # <--- SET THIS (for ArduPilot).
	remote_default_user="debian"  # Mentioned only for reference. Do not change.
	remote_default_user_old_pwd="temppwd"  # Be ready to enter this when the script is run. Do not change.
	remote_ip="192.168.7.2"  # Private or public IP as appropriate.
	remote_root_old_pwd="root"  # Mentioned only for reference. Do not change.
	remote_root_new_pwd="<new password for root>"  # <--- SET THIS.
	remote_default_user_new_pwd="<new password for debian user>"  # <--- SET THIS.
	remote_old_hostname="beaglebone"  # Mentioned only for reference. Do not change.
	remote_new_hostname="<drone's name>"  # <--- SET THIS.
	remote_service_user="<service username>"  # <--- SET THIS.
	remote_service_user_pwd="<service user's password>"  # <--- SET THIS.
	remote_uses_SSID="<WiFi SSID drone will connect to>"  # <--- SET THIS.
	remote_uses_pwd="<WiFi password>"  # <--- SET THIS.
	# User configuration ends here.


	rm ~/.ssh/known_hosts
	local_pub_key="$(cat /etc/ssh/ssh_host_ecdsa_key.pub)"
	entry="$local_ip $local_pub_key"
	echo "$entry" >~/known
	ssh-copy-id $remote_default_user@$remote_ip
	scp ~/{.bash_profile,.bashrc,known} ~/{arducopter,arduplane,ardurover,antennatracker} /etc/DIR_COLORS 	$remote_default_user@$remote_ip:~
	cat <<-EOF5 >tmp1
		ssh-keygen -C 'a&b.c' -b 4096 <<-EOF10  # Note that the three blank lines that follow are intentional!



			EOF10
		sudo -kSs <<-EOF15
			$remote_default_user_old_pwd
			echo $remote_new_hostname >/etc/hostname
			echo -e "127.0.1.1\t$remote_new_hostname.localdomain\t$remote_new_hostname" >>/etc/hosts
			echo "$remote_default_user ALL=(ALL) NOPASSWD: ALL" >>/etc/sudoers.d/$remote_default_user
			echo "$remote_service_user ALL=(ALL) NOPASSWD: ALL" >>/etc/sudoers.d/$remote_service_user
			:>| /etc/motd
			echo "root:$remote_root_new_pwd" | chpasswd
			echo "$remote_default_user:$remote_default_user_new_pwd" | chpasswd
			useradd -m -G sudo -s /bin/bash $remote_service_user
			echo "$remote_service_user:$remote_service_user_pwd" | chpasswd
			mv /home/$remote_default_user/DIR_COLORS /etc
			cp /home/$remote_default_user/.bash_profile /root
			cp /home/$remote_default_user/.bashrc /root
			cp /home/$remote_default_user/.bash_profile /home/$remote_service_user
			cp /home/$remote_default_user/.bashrc /home/$remote_service_user
			# cp /home/$remote_default_user/known /root/.ssh/known_hosts
			cp /home/$remote_default_user/known /home/$remote_default_user/.ssh/known_hosts
			mkdir -p /home/$remote_service_user/.ssh
			chmod 0700 /home/$remote_service_user/.ssh
			cp /home/$remote_default_user/known /home/$remote_service_user/.ssh/known_hosts
			cp /home/$remote_default_user/.ssh/authorized_keys /home/$remote_service_user/.ssh
			chown root:root /etc/DIR_COLORS
			chown $remote_default_user:$remote_default_user /home/$remote_default_user/.bash_profile
			chown $remote_default_user:$remote_default_user /home/$remote_default_user/.bashrc
			chown $remote_service_user:$remote_service_user /home/$remote_service_user/.bash_profile
			chown $remote_service_user:$remote_service_user /home/$remote_service_user/.bashrc
			chown $remote_default_user:$remote_default_user /home/$remote_default_user/.ssh/known_hosts
			chown $remote_service_user:$remote_service_user /home/$remote_service_user/.ssh /home/$remote_service_user/.ssh/known_hosts
			chown $remote_service_user:$remote_service_user /home/$remote_service_user/.ssh/authorized_keys
			rm /home/$remote_default_user/known
			mkdir -p /usr/bin/ardupilot
			mv /home/$remote_default_user/{arducopter,arduplane,ardurover,antennatracker} /usr/bin/ardupilot
			chmod 0755 /usr/bin/ardupilot/a*
			echo "*/5 * * * * echo \"\\\\\\\$(hostname)    \\\\\\\$(id -un)    \\\\\\\$(date)    \\\\\\\$(ip route get 8.8.8.8 | awk '{print \\\\\\\$7; exit}')    \\\\\\\$(dig +short myip.opendns.com @resolver1.opendns.com)\" >~/$remote_default_user@$remote_new_hostname && scp ~/$remote_default_user@$remote_new_hostname $local_user@$local_ip:~" | tee -a /var/spool/cron/crontabs/$remote_default_user >/dev/null
			echo "*/5 * * * * echo \"\\\\\\\$(hostname)    \\\\\\\$(id -un)    \\\\\\\$(date)    \\\\\\\$(ip route get 8.8.8.8 | awk '{print \\\\\\\$7; exit}')    \\\\\\\$(dig +short myip.opendns.com @resolver1.opendns.com)\" >~/$remote_service_user@$remote_new_hostname && scp ~/$remote_service_user@$remote_new_hostname $local_user@$local_ip:~" | tee -a /var/spool/cron/crontabs/$remote_service_user >/dev/null
			chgrp crontab /var/spool/cron/crontabs/$remote_default_user /var/spool/cron/crontabs/$remote_service_user
			chmod 0600 /var/spool/cron/crontabs/$remote_default_user /var/spool/cron/crontabs/$remote_service_user
			cat <<-EOF20 >/var/lib/connman/wifi.config
				[service_\\\$(connmanctl services | grep '$remote_uses_SSID' | grep -Po 'wifi_[^ ]+')]
				Type = wifi
				Security = wpa2
				Name = $remote_uses_SSID
				Passphrase = $remote_uses_pwd
				EOF20
			sleep 5
			while true; do
				if ping -c 1 google.com; then
					echo "Internet connection established."
					break
				else
					echo "Connecting.."; sleep 2
				fi
			done
			DEBIAN_FRONTEND=noninteractive apt-get -y update
			sleep 5
			DEBIAN_FRONTEND=noninteractive apt-get -y dist-upgrade
			sleep 5
			DEBIAN_FRONTEND=noninteractive apt-get install -y cpufrequtils git
			sleep 5
			DEBIAN_FRONTEND=noninteractive apt-get install -y gawk dnsutils
			sleep 5
			# DEBIAN_FRONTEND=noninteractive apt-get install -y flite
			# sleep 5
			cat <<-EOF25 >/etc/default/ardupilot
				TELEM1="-C /dev/ttyO1"
				TELEM2="-A $local_protocol:$local_ip:$local_port"
				GPS="-B /dev/ttyS2"
				EOF25
			cat <<-EOF30 >/lib/systemd/system/arducopter.service
				[Unit]
				Description=ArduCopter Service
				After=networking.service
				StartLimitIntervalSec=0
				Conflicts=arduplane.service ardurover.service antennatracker.service

				[Service]
				EnvironmentFile=/etc/default/ardupilot
				ExecStartPre=/usr/bin/ardupilot/aphw
				ExecStart=/usr/bin/ardupilot/arducopter $TELEM1 $TELEM2 $GPS

				Restart=on-failure
				RestartSec=1

				[Install]
				WantedBy=multi-user.target
				EOF30
			cat <<-EOF35 >/lib/systemd/system/arduplane.service
				[Unit]
				Description=ArduPlane Service
				After=networking.service
				StartLimitIntervalSec=0
				Conflicts=arducopter.service ardurover.service antennatracker.service

				[Service]
				EnvironmentFile=/etc/default/ardupilot
				ExecStartPre=/usr/bin/ardupilot/aphw
				ExecStart=/usr/bin/ardupilot/arduplane $TELEM1 $TELEM2 $GPS

				Restart=on-failure
				RestartSec=1

				[Install]
				WantedBy=multi-user.target
				EOF35
			cat <<-EOF40 >/lib/systemd/system/ardurover.service
				[Unit]
				Description=ArduRover Service
				After=networking.service
				StartLimitIntervalSec=0
				Conflicts=arducopter.service arduplane.service antennatracker.service

				[Service]
				EnvironmentFile=/etc/default/ardupilot
				ExecStartPre=/usr/bin/ardupilot/aphw
				ExecStart=/usr/bin/ardupilot/ardurover $TELEM1 $TELEM2 $GPS

				Restart=on-failure
				RestartSec=1

				[Install]
				WantedBy=multi-user.target
				EOF40
			cat <<-EOF45 >/lib/systemd/system/antennatracker.service
				[Unit]
				Description=AntennaTracker Service
				After=networking.service
				StartLimitIntervalSec=0
				Conflicts=arducopter.service arduplane.service ardurover.service

				[Service]
				EnvironmentFile=/etc/default/ardupilot
				ExecStartPre=/usr/bin/ardupilot/aphw
				ExecStart=/usr/bin/ardupilot/antennatracker $TELEM1 $TELEM2 $GPS

				Restart=on-failure
				RestartSec=1

				[Install]
				WantedBy=multi-user.target
				EOF45
			cat <<-EOF50 >/usr/bin/ardupilot/aphw
				#!/bin/bash
				# aphw
				# ArduPilot hardware configuration.

				/bin/echo 80 >/sys/class/gpio/export
				/bin/echo out >/sys/class/gpio/gpio80/direction
				/bin/echo 1 >/sys/class/gpio/gpio80/value
				/bin/echo pruecapin_pu >/sys/devices/platform/ocp/ocp:P8_15_pinmux/state
				EOF50
			chmod 0755 /usr/bin/ardupilot/aphw
			cat <<-EOF55 | tee /home/$remote_default_user/ap-start >/home/$remote_service_user/ap-start
				#!/bin/bash
				# ap-start
				# Starts ArduPilot manually.

				sudo /usr/bin/ardupilot/arducopter -C /dev/ttyO1 -A udp:192.168.0.41:14550 -B /dev/ttyS2 &
				EOF55
			chmod 0755 /home/$remote_default_user/ap-start /home/$remote_service_user/ap-start
			chown $remote_default_user:$remote_default_user /home/$remote_default_user/ap-start
			chown $remote_service_user:$remote_service_user /home/$remote_service_user/ap-start
			cat <<-EOF60 | tee /home/$remote_default_user/ap-stop >/home/$remote_service_user/ap-stop
				#!/bin/bash
				# ap-stop
				# Stops ArduPilot manually.

				sudo systemctl stop arducopter.service
				EOF60
			chmod 0755 /home/$remote_default_user/ap-stop /home/$remote_service_user/ap-stop
			chown $remote_default_user:$remote_default_user /home/$remote_default_user/ap-stop
			chown $remote_service_user:$remote_service_user /home/$remote_service_user/ap-stop
			cat <<-EOF65 | tee /home/$remote_default_user/ap-clone >/home/$remote_service_user/ap-clone
				#!/bin/bash
				# ap-clone
				# Clones ArduPilot and installs rangerfinder firmware.

				git clone https://github.com/ArduPilot/ardupilot.git
				cd ardupilot/Tools/Linux_HAL_Essentials/pru/rangefinderpru
				sudo make install
				EOF65
			chmod 0755 /home/$remote_default_user/ap-clone /home/$remote_service_user/ap-clone
			chown $remote_default_user:$remote_default_user /home/$remote_default_user/ap-clone
			chown $remote_service_user:$remote_service_user /home/$remote_service_user/ap-clone
			cat <<-EOF70 | tee /home/$remote_default_user/ap-build >/home/$remote_service_user/ap-build
				#!/bin/bash
				# ap-build
				# Builds ArduPilot.

				echo "This is going to take some time.."
				cd ardupilot
				git checkout Copter-3.5.5
				# git checkout ArduPlane-3.8.4
				git submodule update --init --recursive
				./waf configure --board=blue
				./waf
				sudo cp ./build/blue/bin/a* /usr/bin/ardupilot
				EOF70
			chmod 0755 /home/$remote_default_user/ap-build /home/$remote_service_user/ap-build
			chown $remote_default_user:$remote_default_user /home/$remote_default_user/ap-build
			chown $remote_service_user:$remote_service_user /home/$remote_service_user/ap-build
			cat <<-EOF75 | tee /home/$remote_default_user/rt-kernel >/home/$remote_service_user/rt-kernel
				#!/bin/bash
				# rt-kernel
				# Install real-time kernel.

				/opt/scripts/tools/update_kernel.sh --ti-rt-channel --lts-4_9
				# /opt/scripts/tools/update_kernel.sh --ti-rt-channel --lts-4_14
				reboot
				EOF75
			chmod 0755 /home/$remote_default_user/rt-kernel /home/$remote_service_user/rt-kernel
			chown $remote_default_user:$remote_default_user /home/$remote_default_user/rt-kernel
			chown $remote_service_user:$remote_service_user /home/$remote_service_user/rt-kernel
			cat <<-EOF80 >/etc/network/interfaces
				# The loopback network interface.
				auto lo
				iface lo inet loopback

				# # WiFi w/ onboard device (dynamic IP).
				# auto wlan0
				# iface wlan0 inet dhcp
				# wpa-ssid "$remote_uses_SSID"
				# wpa-psk "$remote_uses_pwd"

				# # WiFi w/ onboard device (static IP).
				# auto wlan0
				# iface wlan0 inet static
				# wpa-ssid "$remote_uses_SSID"
				# wpa-psk "$remote_uses_pwd"
				# address 192.168.0.99  # <--- The desired static IP address of the BBBlue.
				# netmask 255.255.255.0
				# gateway 192.168.0.1  # <--- The address of your router.

				# Ethernet/RNDIS gadget (g_ether).
				# Used by: /opt/scripts/boot/autoconfigure_usb0.sh
				iface usb0 inet static
				address 192.168.7.2
				netmask 255.255.255.252
				network 192.168.7.0
				gateway 192.168.7.1

				# # WiFi AP w/ USB adapter (thanks to Patrick Poirier).
				# auto wlan1
				# allow-hotplug wlan1
				# iface wlan1 inet static
				# hostapd /etc/hostapd/hostapd.conf
				# address 192.168.8.1
				# netmask 255.255.255.0
				EOF80
			cat <<-EOF85 >/etc/hostapd/hostapd.conf
				interface=wlan1
				driver=nl80211
				ssid=BBBlue-AP
				channel=1
				ignore_broadcast_ssid=0
				country_code=US
				ieee80211d=1
				hw_mode=g
				macaddr_acl=0
				max_num_sta=10
				auth_algs=1
				ignore_broadcast_ssid=0
				rsn_preauth=1
				rsn_preauth_interfaces=wlan1
				wpa=3
				wpa_passphrase=123456789
				wpa_key_mgmt=WPA-PSK
				wpa_pairwise=TKIP
				rsn_pairwise=CCMP
				EOF85
			cat <<-EOF90 >/etc/dnsmasq.conf
				interface=lo,wlan1
				no-dhcp-interface=lo
				dhcp-range=192.168.8.20,192.168.8.40,255.255.255.0,12h

				interface=usb0
				dhcp-range=192.168.7.1,192.168.7.1,12h
				EOF90
			cat <<-EOF95 >/etc/network/interfaces2
				allow-hotplug wlan1
				auto wlan1
				iface wlan1 inet static
				hostapd /etc/hostapd/hostapd.conf
				address 192.168.8.1
				netmask 255.255.255.0
				EOF95
			cat <<-EOF100 | tee /home/$remote_default_user/start-wifi >/home/$remote_service_user/start-wifi
				#!/bin/bash
				# start-wifi
				# Starts WiFi (using connman).

				echo "Starting WiFi.."
				connmanctl enable wifi &>/dev/null
				while true; do
					if [ "\\\\\\\$(connmanctl state | grep 'State = ' | cut -d ' ' -f 5)" == "online" ]; then
						echo "WiFi up."
						break
					else
						echo "\\\\\\\$(connmanctl state | grep 'State = ' | cut -d ' ' -f 5)"$'..'; sleep 1
					fi
				done
				EOF100
			chmod 0755 /home/$remote_default_user/start-wifi /home/$remote_service_user/start-wifi
			chown $remote_default_user:$remote_default_user /home/$remote_default_user/start-wifi
			chown $remote_service_user:$remote_service_user /home/$remote_service_user/start-wifi
			cat <<-EOF105 | tee /home/$remote_default_user/stop-wifi >/home/$remote_service_user/stop-wifi
				#!/bin/bash
				# stop-wifi
				# Stops WiFi (using connman).

				echo "Stopping WiFi.."
				connmanctl disable wifi &>/dev/null
				while true; do
					if [ "\\\\\\\$(connmanctl state | grep 'State = ' | cut -d ' ' -f 5)" == "idle" ]; then
						echo "WiFi down."
						break
					else
						echo "\\\\\\\$(connmanctl state | grep 'State = ' | cut -d ' ' -f 5)"$'..'; sleep 1
					fi
				done
				EOF105
			chmod 0755 /home/$remote_default_user/stop-wifi /home/$remote_service_user/stop-wifi
			chown $remote_default_user:$remote_default_user /home/$remote_default_user/stop-wifi
			chown $remote_service_user:$remote_service_user /home/$remote_service_user/stop-wifi
			cat <<-EOF110 | tee /home/$remote_default_user/start-access >/home/$remote_service_user/start-access
				#!/bin/bash
				# start-access
				# Enables WiFi Access Point (wlan1).

				echo "Enabling WiFi Access Point.."
				sudo systemctl start hostapd.service
				EOF110
			chmod 0755 /home/$remote_default_user/start-access /home/$remote_service_user/start-access
			chown $remote_default_user:$remote_default_user /home/$remote_default_user/start-access
			chown $remote_service_user:$remote_service_user /home/$remote_service_user/start-access
			cat <<-EOF115 | tee /home/$remote_default_user/stop-access >/home/$remote_service_user/stop-access
				#!/bin/bash
				# stop-access
				# Disables WiFi Access Point (wlan1).

				echo "Disabling WiFi Access Point.."
				sudo systemctl stop hostapd.service
				EOF115
			chmod 0755 /home/$remote_default_user/stop-access /home/$remote_service_user/stop-access
			chown $remote_default_user:$remote_default_user /home/$remote_default_user/stop-access
			chown $remote_service_user:$remote_service_user /home/$remote_service_user/stop-access
			cat <<-EOF120 | tee /home/$remote_default_user/.asoundrc >/home/$remote_service_user/.asoundrc
				pcm.!default {
					type plug
					slave {
						pcm "hw:1,0"
					}
				}

				ctl.!default {
					type hw
					card 1
				}
				EOF120
			chown $remote_default_user:$remote_default_user /home/$remote_default_user/.asoundrc
			chown $remote_service_user:$remote_service_user /home/$remote_service_user/.asoundrc
			cd /opt/scripts && git pull
			/opt/scripts/tools/update_kernel.sh --ti-rt-channel --lts-4_9
			# /opt/scripts/tools/update_kernel.sh --ti-rt-channel --lts-4_14
			sed -i 's/#dtb=/dtb=am335x-boneblue.dtb/g' /boot/uEnv.txt
			sed -i 's/GOVERNOR="ondemand"/GOVERNOR="performance"/g' /etc/init.d/cpufrequtils
			sudo systemctl disable bb-wl18xx-bluetooth.service
			sleep 5
			systemctl enable arducopter.service
			# systemctl enable arduplane.service
			# systemctl enable ardurover.service
			# systemctl enable antennatracker.service
			sleep 5
			# systemctl disable connman.service
			# sleep 5
			# systemctl enable hostapd.service
			# sleep 5
			/opt/scripts/tools/grow_partition.sh
			su - $remote_service_user <<-EOF125
				ssh-keygen -C 'a&b.c' -b 4096 <<-EOF130  # Note that the three blank lines that follow are intentional!



					EOF130
				EOF125
			EOF15
		EOF5
	ssh $remote_default_user@$remote_ip '/bin/bash -s' <tmp1
	rm tmp1


	ssh-copy-id $remote_default_user@$remote_ip
	ssh-copy-id $remote_service_user@$remote_ip
	scp $remote_default_user@$remote_ip:~/.ssh/id_rsa.pub ~
	cat ~/id_rsa.pub >>~/.ssh/authorized_keys
	scp $remote_service_user@$remote_ip:~/.ssh/id_rsa.pub ~
	cat ~/id_rsa.pub >>~/.ssh/authorized_keys
	rm ~/id_rsa.pub
	rm ~/known


	cat <<-EOF135 >tmp2
		sudo -kSs <<-EOF140  # Added $remote_default_user & $remote_service_user to sudoers, so no password needed this time.
			# userdel -rf debian  # But I'm logging in as $remote_default_user at the moment..
			sed -i 's/#cmdline=init=/cmdline=init=/g' /boot/uEnv.txt  # Comment out to prevent flashing to eMMC.
			echo -e "\nRebooting remote device. Press Ctrl-Z if connection hangs."
			reboot
			EOF140
		EOF135
	ssh $remote_default_user@$remote_ip '/bin/bash -s' <tmp2
	rm tmp2

-- Imf
