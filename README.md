#

My continuously updated (as of 24/04/2018 UK date format) beginner's guide for setting up the BeagleBone Blue for Mirko Denecke's port of ArduPilot (https://github.com/mirkix/ardupilotblue). Only necessary steps are indicated here. Information found elsewhere may indicate extra steps that are no longer necessary due to the software having been updated.

1) Go to https://rcn-ee.net/rootfs/bb.org/testing/ and select the directory named with the latest date. Then click on the stretch-console directory. You'll see a number of files here. Download the file named something like bone-debian-9.X-console-armhf-20YY-MM-DD-1gb.img.xz. This is what's known as the 'console' image. This is a very minimal image of Debian - it contains only the bare essentials. An alternative is the 'IoT' image (IoT = Internet of Things) which comes with additional software and can make for a more comfortable experience if you are very new to Linux. It's available on the same website.

2) Now you will need to copy the image to a microSD card. Whether are you are using Linux or Windows, I highly recommend a program called Etcher for this task. It's very easy to use and it works!

