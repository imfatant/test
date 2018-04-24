My continuously updated (as of 24/04/2018 UK date format) beginner's guide for setting up the BeagleBone Blue with Mirko Denecke's port of ArduPilot (https://github.com/mirkix/ardupilotblue).

Only necessary steps are indicated here. Information found elsewhere may indicate extra steps that are no longer necessary due to the software having been updated.

1) Go to https://rcn-ee.net/rootfs/bb.org/testing/ and select the directory named with the latest date. Then click on the stretch-console directory. You'll see a number of files here. Download the file named something like bone-debian-V.V-console-armhf-20YY-MM-DD-1gb.img.xz. This is what's known as the 'console' image. This is a very minimal image of Debian - it contains only the bare essentials. An alternative is the 'IoT' image (IoT = Internet of Things) which comes with additional software and can make for a more comfortable experience if you are very new to Linux. It's available on the same website.

2) Now you will need to copy the image to a microSD card. Whether are you are using Linux or Windows, I highly recommend a program called Etcher for this task (https://etcher.io/). It's very easy to use and it works!

3) It should now be possible to boot up the BeagleBone Blue from the microSD card. It is beyond the scope of this document to detail ways of interacting with the BBBlue, but often it's accomplished by plugging in a Micro-USB cable and either using SSH (to 'debian@192.168.7.2', password 'temppwd') or establishing a serial link over a COM port (user 'debian', password 'temppwd)'. More information can be found here: https://beagleboard.org/blue.

4) Hopefully, you now find yourself logged into the debian user account and at the command prompt. The next task is to update and install some software using an available internet connection. Usually, at this stage, you'll share your computer's internet connection with the BBBlue, and so you must tell the Blue about this.
  If you are SSHing to 192.168.7.2, then type (at the BBlue's command prompt): `sudo /sbin/route add default gw 192.168.7.1`
  If you are using a serial link, ...

