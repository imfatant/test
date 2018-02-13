Cross-compiling ArduCopter Blue on Arch Linux x64 (and using pacaur instead of yaourt):

gpg --recv-keys 79BE3E4300411886 38DBBDC86092693E 79C43DFBF1CF2187
pacaur -S arm-linux-gnueabihf-gcc  # <--- Note that I'm using pacaur instead of yaourt. Also, ensure you haven't set any C/C++ env variables.
sudo ln -s pkg-config /usr/bin/arm-linux-gnueabihf-pkg-config
sudo pip install future
git clone https://github.com/diydrones/ardupilot.git
cd ardupilot
git config user.name <your username goes here>
sed -i 's/command -v yaourt/command -v pacaur/g' ./Tools/scripts/install-prereqs-arch.sh  # <--- Skip this if using yaourt.
sed -i 's/yaourt -S --noconfirm --needed/pacaur -S --noconfirm --noedit/g' ./Tools/scripts/install-prereqs-arch.sh  # <--- Skip if using yaourt.
git commit -a --allow-empty-message -m ''  # <--- The lazy option. Skip if using yaourt.
./Tools/scripts/install-prereqs-arch.sh
git checkout Copter-3.5.5  # I'm only wanting ArduCopter right now.
git submodule update --init --recursive
./waf configure --board=blue  # <--- BeagleBone Blue for me.
./waf copter -j2
scp ./build/blue/bin/* debian@192.168.0.24:/home/debian  # <--- Finally, copy the compiled executable(s) over to the BBone.
