live-cd2
========

[![CircleCI](https://circleci.com/gh/tork-a/live-cd2.svg?style=svg)](https://circleci.com/gh/tork-a/live-cd2)

This is 2nd generaiton of live-cd

Contents
--------
```
  README.md  This file
  Makefile   Commands to create live-cd
  indigo/
      Dockerfile : Docker images to be installed in live-cd
  vagrant/
      Vargantfile: Commands to setup vargant
  builder/
      Makefile   : Commands to build iso from Docker using vagrant
```
Burning
-------

<!--

  - Use gparted and create partitions:

     - create fat32 3686 MB  (3.6G)

     - create ext4 with casper-rw label (for the rest spaces)

  - then burn iso using `usb-creator-gtk` with enabling persistent
  (http://askubuntu.com/questions/138356/how-do-i-get-a-live-usb-to-use-a-partition-for-persistence/181062#181062)

  - Boot burned usb and run `sudo update-initramfs -u` and shotdown, better to disable wifi (looking for wifi goes black screen during boot up)
  
    - You can run burned sub in virtual machine : `sudo kvm --hdb  /dev/sdb -m 2G`

    - Note : 14.04 has bugs on casper-src, you need  (https://github.com/tork-a/live-cd2/commit/0c1e155a803697d94fc630de3ebd512ca75d5bfe)
      ```
    wget http://archive.ubuntu.com/ubuntu/pool/main/c/casper/casper_1.360_amd64.deb
    dpkg -x casper_1.360_amd64.deb casper-src
    cp casper-src/usr/share/initramfs-tools/scripts/casper /usr/share/initramfs-tools/scripts/casper
    update-initramfs -u
      ```

  - Insert the USB and remove `casper-rw` file from you fat32 pertition

-->


  - Use mkusb to burn persistent live cd:

    - Note that you have to use BIOS (not UEFI) booted computer to start `mkusbe`

    - http://askubuntu.com/questions/397481/how-to-make-a-persistent-live-ubuntu-usb-with-more-than-4gb/753163#753163

     - `sudo add-apt-repository ppa:mkusb/ppa  # and press Enter`
     - `sudo apt-get update`
     - `sudo apt-get install mkusb usb-pack-efi`
     - `sudo -H mkusb indigo-tork-*.iso` # source file and persistence via parameters
     - 1. Choose `d:     dus , guidus, mkusb-dus    - New, easy to use`
     - 2. Choose `p      'Persistent live' - only Debian and Ubuntu`
     - 3. Choose `sdb    Generic_Flash_Disk`
     - 4. Choose `upefi  usb-pack-efi    (default grub from ISO file)`
     - Choose 80% for persistent space on 8G memory will make 1.0G free pertition on Windows and 4.0G persitent disk strage for Ubuntu
     - https://askubuntu.com/questions/693719/how-to-boot-from-a-usb-drive-in-virtualbox

References:
- http://github.com/mbentley/overclockix
- http://github.com/znz/rubylive
- http://blog.n-z.jp/blog/2015-10-26-rubylive-builder.html
- http://docs.kali.org/development/live-build-a-custom-kali-iso
- http://github.com/debian-live/live-build
- http://yakushi.shinshu-u.ac.jp/robotics/?DebianLive%2FBuild-HOWTO
