# ArduPilot Blue - a beginner's guide
This is my continuously updated (as of 26/04/2018, DD/MM/20YY) beginner's guide to setting up the BeagleBone Blue with Mirko Denecke's port of ArduPilot (https://github.com/mirkix/ardupilotblue). This guide is based in great part upon similar documents at Patrick Poirier's PocketPilot project (https://github.com/PocketPilot/PocketPilot). Many thanks to both these fine individuals, and others too @ https://gitter.im/mirkix/BBBMINI, for their superb work!

I take a minimalistic approach. Only necessary steps are shown, excepting that I install Git for the sake of convenience. Information found elsewhere may indicate steps that are no longer necessary due to the software having been updated, or steps that only apply to other platforms (BBBMINI or PocketPilot, etc).

# Part 1
0) Before I begin, I want to stress that supplying the BeagleBone Blue with adequate power is a must. Typically, we'll be attaching quite a few navigation-related peripherals to it, and they'll not behave correctly without enough juice.

1) Go to https://rcn-ee.net/rootfs/bb.org/testing/ and select the directory named with the latest date. Then click on the stretch-console subdirectory. You'll see a number of files here. Download the file named something like 'bone-debian-V.V-console-armhf-20YY-MM-DD-1gb.img.xz'. This is what's known as the 'console' image. It's a very minimal distribution of Debian with only the bare essentials. An alternative is the 'IoT' image (IoT = Internet of Things) which comes with additional software and can make for a more comfortable experience if you are very new to Linux. It's available from the same site.

    Don't assume that the latest image is the best, or even that it works. Afterall, this is the 'testing' repository, so be warned. However, I will endeavour to keep abreast of what seems to be the latest functional image and mention it here.

    At the time of writing, I'm using https://rcn-ee.net/rootfs/bb.org/testing/2018-04-22/stretch-console/bone-debian-9.4-console-armhf-2018-04-22-1gb.img.xz
    
2) Now you'll need to flash the image to a microSD card. Whether you are using Linux or Windows, I highly recommend a program called Etcher for this task (https://etcher.io/).

3) It should now be possible to boot up the BeagleBone Blue from the microSD card. It's beyond the scope of this document to detail all the ways of interacting with the BBBlue, but often it's accomplished by plugging in a Micro-USB cable and either using SSH (to 'debian@192.168.7.2', password 'temppwd') or establishing a serial link over a COM port in a program like Minicom or PuTTY (user 'debian', password 'temppwd)'. More information can be found here: https://beagleboard.org/blue.

    BeagleBone drivers come with Windows 10, but Linux distributions can be fiddly. If you're experiencing problems with Linux, do this:

       sudo -s
       cat >/etc/udev/rules.d/73-beaglebone.rules <<EOF
       ACTION=="add", SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_interface", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="a6d0", DRIVER=="", RUN+="/sbin/modprobe -b ftdi_sio"
       ACTION=="add", SUBSYSTEM=="drivers", ENV{DEVPATH}=="/bus/usb-serial/drivers/ftdi_sio", ATTR{new_id}="0403 a6d0"
       ACTION=="add", KERNEL=="ttyUSB*", ATTRS{interface}=="BeagleBone", ATTRS{bInterfaceNumber}=="00", SYMLINK+="beaglebone-jtag"
       ACTION=="add", KERNEL=="ttyUSB*", ATTRS{interface}=="BeagleBone", ATTRS{bInterfaceNumber}=="01", SYMLINK+="beaglebone-serial"
       EOF
       udevadm control --reload-rules
       exit

4) Hopefully, you now find yourself logged in to the debian user account and at the command prompt. The next task is to update and install some software using an available internet connection.

    You can tell the BBBlue that it'll be sharing your computer's internet connection by typing (at the BBBlue's command prompt):
    
       sudo /sbin/route add default gw 192.168.7.1
       echo "nameserver 8.8.8.8" | sudo tee -a /etc/resolv.conf >/dev/null
    You can then tell your Linux computer to share with the BBBlue by typing (at the computer's command prompt):
    
       sudo sysctl net.ipv4.ip_forward=1
       sudo iptables -t nat -A POSTROUTING -o eno1 -j MASQUERADE
       sudo iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
       sudo iptables -A FORWARD -i enp0s20u7 -o eno1 -j ACCEPT
    On my Arch Linux desktop x64 PC, 'eno1' is the name of the Ethernet adapter connected to my router (and the connection I'll be sharing), and 'enp0s20u7' is the name the OS shows for the BBBlue. Use `ip link` to find the corresponding names for your devices.
    
    If you are using a Windows computer, you can tell it to share with the BBBlue by fast-forwarding to about 5:00 of this Derek Molloy video: http://derekmolloy.ie/beaglebone/getting-started-usb-network-adapter-on-the-beaglebone/.
    
    If, for whatever reason, internet sharing isn't an option, another possibility is a USB-to-Ethernet dongle like this one: http://accessories.ap.dell.com/sna/productdetail.aspx?c=sg&l=en&s=bsd&cs=sgbsd1&sku=470-ABNL). One can plug this into the BBBlue and then connect it directly to the router. Note that, in my case, I had to supply extra power to the BBBlue via its 2s LiPo connector or its jack plug for the dongle to work. The BBBlue enumerates the dongle as device 'usb2', and sets it up automatically (well, sometimes it's necessary to reboot in order for `ip link` to show that the device is up).
    
    "But wait!", you may be thinking, "Surely I can just use the BBBlue's onboard WiFi fucntionality to connect to my router and get internet access that way?" Absolutely, you can, but that's beacause you've used the IoT image, which has all the software necessary to do this out of the box. The downside is that it also contains a lot of superfluous stuff. If you're happy with this, skip ahead to Steps 13 and 14, and then do Steps 5 through to 12.
    
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
13) It's time to set up connman for WiFi access. I do it the following way because it's easier to automate in a script later on. First, make a note of your router's SSID and WiFi password. Then type the following:

        sudo -s
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
    Finally, type: `systemctl enable connman`

14) Reboot again now: `reboot`

# Part 2
15) When the BBBlue comes back up, you should notice that a prominent green LED is lit, signifying that WiFi is working. The BBBlue is connected to your router, and its address on your WiFi network can be found using programs like nmap: `sudo nmap 192.168.0.0/24`. It's also possible that you can find the address by logging in to your router and looking there. Try SSHing to the BBBlue using its WiFi IP address. 192.168.7.2 will still work as well.

    By the way, if you used a USB-to-Ethernet dongle in Step 4, you may need to unplug it now, and hopefully your green LED will go on, too.

16) Now we need to create a few text files. Use your favourite text editor (with sudo). Personally, I like nano. First, the ArduPilot environment configuration file, /etc/default/ardupilot:
        
        TELEM1="-C /dev/ttyO1"
        TELEM2="-A udp:192.168.0.13:14550"
        GPS="-B /dev/ttyS2"
    This is a pretty typical config. It breaks down like this:
    
    Switch -C maps ArduPilot's "Telem1" serial port (SERIAL1, default 57600) to the BBBlue's UART1. For example, I have a RFDesign 868x radio modem connected to UART1. It is the bidirectional datalink with my drone. It sends various telemetry data to the base station, and receives commands and RTK differential corrections from the base station.
    
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
    
17) Next, we'll create the ArduPilot systemd service files, one for ArduPlane, /lib/systemd/system/arduplane.service:
    
        [Unit]
        Description=ArduPlane Service
        After=networking.service
        StartLimitIntervalSec=0
        Conflicts=arducopter.service ardurover.service ardutracker.service

        [Service]
        EnvironmentFile=/etc/default/ardupilot
        ExecStartPre=/usr/bin/ardupilot/aphw
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
        Conflicts=arduplane.service ardurover.service ardutracker.service

        [Service]
        EnvironmentFile=/etc/default/ardupilot
        ExecStartPre=/usr/bin/ardupilot/aphw
        ExecStart=/usr/bin/ardupilot/arducopter $TELEM1 $TELEM2 $GPS

        Restart=on-failure
        RestartSec=1

        [Install]
        WantedBy=multi-user.target
18) Pause for a moment to create a directory: `sudo mkdir -p /usr/bin/ardupilot`
    
    Then, carry on with creating what I call the ArduPilot hardware configuration file, /usr/bin/ardupilot/aphw, which is run by the services prior to running the ArduPlane or ArduCopter executables:

        #!/bin/bash
        # aphw
        # ArduPilot hardware configuration.

        /bin/echo 80 >/sys/class/gpio/export
        /bin/echo out >/sys/class/gpio/gpio80/direction
        /bin/echo 1 >/sys/class/gpio/gpio80/value
        /bin/echo pruecapin_pu >/sys/devices/platform/ocp/ocp:P8_15_pinmux/state
    You may want to use `sudo chmod 0744 /usr/bin/ardupilot/aphw` to set permissions for this file.
    
19) Almost there! You must now obtain the latest ArduPlane and ArduCopter executables, built specifically for the BBBlue's Arm architecture, and place them in the /usr/bin/ardupilot directory. Depending on your situation, this may mean building them from scratch. Do not be intimidated - this is not too difficult. Plus it means you'll be able to build your own ArduPilot software whenever it's updated.

    Compiling them on the BBBlue itself is an option, but takes an absolute age. Patrick explains the process for the BBBMINI (based on a BeagleBone Black) here: https://github.com/mirkix/BBBMINI/blob/master/doc/software/software.md. Fortunately, he also provides instructions to cross-compile them on a relatively powerful desktop x64 PC in Ubuntu, which is much, much faster. Here though, I will run through the process of cross-compiling them in Arch Linux (which happens to be God's Own Linux Distro):

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

# Part 3
Is coming soon.. Also a revision to include ArduRover and ArduTracker!

-- Imf
