My continuously updated (as of 24/04/2018, UK date format) beginner's guide for setting up the BeagleBone Blue with Mirko Denecke's port of ArduPilot (https://github.com/mirkix/ardupilotblue). This guide is based upon a similar one at Patrick Poirier's PocketPilot project (https://github.com/PocketPilot/PocketPilot). Many thanks to both these guys.

Only necessary steps are shown. Information found elsewhere may indicate extra steps that are no longer necessary due to the software having been updated.

1) Go to https://rcn-ee.net/rootfs/bb.org/testing/ and select the directory named with the latest date. Then click on the stretch-console directory. You'll see a number of files here. Download the file named something like bone-debian-V.V-console-armhf-20YY-MM-DD-1gb.img.xz. This is what's known as the 'console' image. It's a very minimal image of Debian with only the bare essentials. An alternative is the 'IoT' image (IoT = Internet of Things) which comes with additional software and can make for a more comfortable experience if you are very new to Linux. It's available from the same site.

    At the time of writing, I'm using bone-debian-9.4-console-armhf-2018-04-22-1gb.img.xz.

2) Now you'll need to copy the image to a microSD card. Whether are you are using Linux or Windows, I highly recommend a program called Etcher for this task (https://etcher.io/). It's very easy to use and it works.

3) It should now be possible to boot up the BeagleBone Blue from the microSD card. It is beyond the scope of this document to detail ways of interacting with the BBBlue, but often it's accomplished by plugging in a Micro-USB cable and either using SSH (to 'debian@192.168.7.2', password 'temppwd') or establishing a serial link over a COM port (user 'debian', password 'temppwd)'. More information can be found here: https://beagleboard.org/blue.

4) Hopefully, you now find yourself logged into the debian user account and at the command prompt. The next task is to update and install some software using an available internet connection.

    If you are SSHing to 192.168.7.2, you can your share your computer's internet connection with the BBBlue very simply by typing (at the BBBlue's command prompt): `sudo /sbin/route add default gw 192.168.7.1`
    
    If you have established a serial link (in a terminal program like Minicom), the process is more difficult, especially as the console image does not contain connman (for WiFi). In the end, I chose to use a USB-to-Ethernet dongle I had lying around as I could plug this into the BBBlue and then connect it directly to my router. Note that I had to supply extra power to the BBBlue via its 2s LiPo connector for the dongle to work. The BBBlue enumerates it as device 'usb2'. I then typed:
        ip link set dev usb2 up
        ifup usb2
    
    
    
        cat <<EOF2 >/etc/network/interfaces
        # The loopback network interface.
        auto lo
        iface lo inet loopback
        
