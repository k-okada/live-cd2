##
## assuming we already have vagrant running docker
##

ifneq ("$(wildcard ../color.mk)","") # http://stackoverflow.com/questions/5553352/how-do-i-check-if-file-exists-in-makefile
include ../color.mk
endif

# build iso, assuing vagrant/docker is up, and docker has been built
all: all-vagrant

all-local: clean setup-local build-local

all-vagrant: clean setup-vagrant build-vagrant

all-docker: clean setup-docker build-docker # not working?

### DEBUG change iso-build to iso-build-for for DBEUG
BUILD_TARGET=iso-build
#BUILD_TARGET=iso-build-debug
iso-build-debug:
	# install
	$(call colorecho, "install ubuntu-defaults-builder build-essential")
	sudo apt-get install -y ubuntu-defaults-builder build-essential xorriso
	# setup
	$(call colorecho, "[DEBUG] template tork-defaults files")
	ubuntu-defaults-template tork-defaults
	# build
	$(call colorecho, "template tork-defaults files")
	cd tork-defaults; dpkg-buildpackage -us -uc
	## ubuntu-defaults-image
	$(call colorecho, "build tork-defaults package")
	sudo ubuntu-defaults-image --package tork-defaults_0.1_all.deb --components main,restricted,universe,multiverse || echo "ok" # --ppa xorg-edgers/ppa

	# run twice http://blog.n-z.jp/blog/2014-07-13-build-rubylive-on-vm.html
	$(call colorecho, "build tork-defaults package again")
	[ -e binary.hybrid.iso ] || DISPLAY= sudo PROJECT=ubuntu ARCH=`dpkg-architecture -qDEB_HOST_ARCH` lb build

	# https://lists.ubuntu.com/archives/ubuntu-devel/2014-November/038540.html
	$(call colorecho, "EFI hack")
	./make-efi-iso.sh

###
###
###
setup-local:
	sudo apt-get install -y git build-essential fakeroot debhelper gdebi live-build ubuntu-defaults-builder p7zip-full
	7z x -y -o/tmp/iso ~/Downloads/ubuntu-14.04.4-desktop-amd64.iso
	make clean

build-local:
	make ${BUILD_TARGET} | tee ${BUILD_TARGET}.log

###
###
###

VAGRANT_HOSTNAME=$(shell cd ../vagrant/;vagrant ssh-config $$VM_TYPE --host vagrant | grep HostName | sed 's@^.*HostName @@')
ifeq (${VM_TYPE},ec2)
    VAGRANT_USERNAME=ubuntu
else
    VAGRANT_USERNAME=vagrant
endif

setup-vagrant:
	$(call colorecho, "docker info on host machine")
	docker -H tcp://:2375 info
	$(call colorecho, "check if vagrant is available")
	sshpass -p vagrant ssh -o StrictHostkeyChecking=no -p 22 ${VAGRANT_USERNAME}@${VAGRANT_HOSTNAME} ls -al
	$(call colorecho, "docker info on vagrant machine")
	sshpass -p vagrant ssh -o StrictHostkeyChecking=no -p 22 ${VAGRANT_USERNAME}@${VAGRANT_HOSTNAME} docker info
	$(call colorecho, "docker images on vagrant machine")
	sshpass -p vagrant ssh -o StrictHostkeyChecking=no -p 22 ${VAGRANT_USERNAME}@${VAGRANT_HOSTNAME} sh -c "uname -a; docker images"
	$(call colorecho, "make lb_build verbose for circleci")
	# make lb build verbose, not to 'took more than 10 minutes since last output'
	sshpass -p vagrant ssh -o StrictHostkeyChecking=no -p 22 ${VAGRANT_USERNAME}@${VAGRANT_HOSTNAME} bash -c "set -x; export TERM=xterm; sudo apt-get -y install live-build; sudo sed -i 's@-no-progress@@' /usr/lib/live/build/lb_binary_rootfs"

build-vagrant: #depends.txt tork-indigo.sh add-apt-repository
	$(call colorecho, "build iso on vagrant machine")
	# location of builder is /opt/builder, see ../vagrant/Vagrantfile config.vm.synced_folder "../builder", "/opt/builder"
	sshpass -p vagrant ssh -o StrictHostkeyChecking=no -p 22 ${VAGRANT_USERNAME}@${VAGRANT_HOSTNAME} bash -c "set -x; uname -a; set -x; hostname; export TERM=xterm; mkdir -p build; cd build; cat /opt/builder/make-efi-iso.sh; cp -r /opt/builder/* ./; ls -al; make ${BUILD_TARGET} VERBOSE=1;"
	sshpass -p vagrant scp -o StrictHostkeyChecking=no -P 22 ${VAGRANT_USERNAME}@${VAGRANT_HOSTNAME}:build/*.iso .

###
###
###
setup-docker:
	$(call colorecho, "build builder")
	docker build -t builder .
	$(call colorecho, "check if builder works")
	docker run -i -t -v ${CURDIR}/..:/opt/ws builder ls -al /opt/
	docker run -i -t -v ${CURDIR}/..:/opt/ws -v /var/run/docker.sock:/var/run/docker.sock builder docker images

build-docker:
	$(call colorecho, "build iso on docker machine")
	docker run -i -t -v ${CURDIR}/..:/opt/ws -v /var/run/docker.sock:/var/run/docker.sock builder make -C /opt/ws/builder/ ${BUILD_TARGET}

###
###
###
update-make-efi-iso-sh:
	sed -i 's@$$(git describe --abbrev=0 --tags)@$(shell git describe --abbrev=0 --tags)@' make-efi-iso.sh

depends.txt:
	$(call colorecho, "get installed packages")
	#docker run --rm tork/indigo bash -c 'echo "sudo apt-get update -qq; dpkg --get-selections | cut -d$$'"'"'\t'"'"' -f1 | xargs -r -n1 bash -c '"'"'apt-cache showpkg \$$0 | grep ubuntu.com > /dev/null && echo \$$0'"'"'" > gen_deps; bash ./gen_deps' | sed s/:amd64// | tee depends.txt
	docker run --rm tork/indigo dpkg --get-selections | sed -e 's@\s*install$$@@g' | sed 's/:amd64//' | grep -v docker-engine | tee depends.txt

tork-indigo.sh:
	$(call colorecho, "get command history")
	# http://stackoverflow.com/questions/19104847/how-to-generate-a-dockerfile-from-an-image
	docker history --no-trunc=true tork/indigo | tr -s " " | sed -e 's/$$ROS_DISTRO/indigo/' | /bin/sed -e 's,#(nop) USER \[\(.*\)\],sudo -u \1 -H -i <<"EOF",' | /bin/sed -e 's,#(nop) WORKDIR \(.*\),cd \1,' | /bin/sed -e 's,#(nop) ENV \(.*\),export \1,' | /bin/sed -n -e 's,[0-9.]* [kMG]\?B *$$,,p' | sed -n -e 's,.*/bin/sh -c \(.*\),\1,p' | /bin/sed -e 's,^\(.*\)#CIRCLECI#,#CIRCLECI# \1,' | tac | sed '0,/MAINTAINER/d' | tee tork-indigo.sh
	echo "EOF" >> tork-indigo.sh

tork-defaults:
	$(call colorecho, "template tork-defaults files")
	ubuntu-defaults-template tork-defaults

	$(call colorecho, "add set -x to chroot")
	echo "set -x" >> tork-defaults/hooks/chroot
	$(call colorecho, "add set -e to chroot")
	echo "set -e" >> tork-defaults/hooks/chroot
	$(call colorecho, "setup i18n/language.txt")
	echo "en" >> tork-defaults/i18n/language.txt
	echo "ja" >> tork-defaults/i18n/language.txt
	$(call colorecho, "setup i18n/langpacks.txt")
	echo "en" >> tork-defaults/i18n/langpacks.txt
	echo "ja complete" >> tork-defaults/i18n/langpacks.txt
	$(call colorecho, "setup i18n/keyboard.txt")
	echo "jp" >> tork-defaults/i18n/keyboard.txt

	$(call colorecho, "add gnome-terminal to unity/launchers.txt")
	echo "gnome-terminal" >> tork-defaults/unity/launchers.txt
	$(call colorecho, "add chromium-browser to unity/launchers.txt")
	echo "chromium-browser" >> tork-defaults/unity/launchers.txt
	$(call colorecho, "copy tork-ros.jpg to desktop/background.jpg")
	cp tork-ros.jpg tork-defaults/desktop/background.jpg

tork-defaults/debian/20_tork-defaults.gschema.override:
	$(call colorecho, "setup 20_tork-defaults.gschema.override")
	echo "[org.gnome.desktop.input-sources]" >> tork-defaults/debian/20_tork-defaults.gschema.override
	$(call colorecho, "gsettings set org.gnome.desktop.input-sources xkb-options ['ctrl:nocaps']")
	echo "xkb-options=['ctrl:nocaps']"  >> tork-defaults/debian/20_tork-defaults.gschema.override

	$(call colorecho, 'gsettings set org.gnome.desktop.input-sources sources [("xkb", "jp") ("ibus", "mozc-jp")]')
	#echo "sources=[('xkb', 'jp'), ('ibus', 'mozc-jp')]" >> tork-defaults/debian/20_tork-defaults.gschema.override
	# custom settings https://bugs.launchpad.net/unity/+bug/1418939
	## I tried to use `gsettings set org.gnome.desktop.input-sources sources [('xkb', 'jp'), ('ibus', 'mozc-jp')]` but it dit not work until you run `gsettings reset org.gnome.desktop.input-sources sources`
	## only `org.gnome.libgnomekbd.keyboard` is working
	echo "[org.gnome.libgnomekbd.keyboard]" >> tork-defaults/debian/20_tork-defaults.gschema.override
	echo "layouts=['jp']" >> tork-defaults/debian/20_tork-defaults.gschema.override

tork-defaults/debian/rc.local:
	$(call colorecho, "setup rc.local")
	cp /etc/rc.local tork-defaults/debian/rc.local
	$(call colorecho, "Generate the SSH keys if non-existent")
	sed -i 's%exit 0%### Generate the SSH keys if non-existent\n[ ! -f /etc/ssh/ssh_host_dsa_key ] \&\& dpkg-reconfigure openssh-server\n\nexit 0%' tork-defaults/debian/rc.local
	$(call colorecho, "Remove tork-defaults.list if exists")
	sed -i 's%exit 0%### Remove tork-defaults.list if exists\n[ -f /etc/apt/sources.list.d/tork-defaults.list ] \&\& rm -f /etc/apt/sources.list.d/tork-defaults.list\n\nexit 0%' tork-defaults/debian/rc.local

tork-defaults/debian/tork-defaults.install: tork-defaults/debian/20_tork-defaults.gschema.override # tork-defaults/debian/rc.local
	$(call colorecho, "setup tork-defaults.install")
	echo "debian/20_tork-defaults.gschema.override usr/share/glib-2.0/schemas"  >> tork-defaults/debian/tork-defaults.install
	#echo "debian/rc.local etc/"  >> tork-defaults/debian/tork-defaults.install

add-apt-repository: archives/tork-defaults.list archives/tork-defaults.key
	# https://lists.debian.org/debian-live/2012/01/msg00008.html says lb2 uses config/chroot_sources but lb3 uses config/archives, but if we only have config files in archives, we have "Some packages could not be installed" error on "P: Begin installing packages (install pass)..."
	# custom repos...

archives/tork-defaults.list:
	$(call colorecho, "get sources.list.d/*.list")
	## get from docker
	mkdir -p archives/
	docker run --rm tork/indigo sh -c 'cat /etc/apt/sources.list.d/*.list' > archives/tork-defaults.list;
	sed -i 's@https://@http://@' archives/tork-defaults.list

archives/tork-defaults.key:
	$(call colorecho, "get apt-keys")
	mkdir -p archives/
	docker run --rm tork/indigo apt-key exportall | gpg --dearmor > archives/tork-defaults.key


iso-build:
	# install
	$(call colorecho, "install ubuntu-defaults-builder build-essential")
	sudo apt-get install -y ubuntu-defaults-builder build-essential xorriso

	$(call colorecho, "make tork-indigo.sh")
	make tork-indigo.sh

	$(call colorecho, "make depends.txt")
	make depends.txt

	### ubuntu-defaults-image
	$(call colorecho, "make tork-defaults")
	make tork-defaults

	# keyboard custom settings
	$(call colorecho, "make tork-defaults.install")
	make tork-defaults/debian/tork-defaults.install

	# custom from Dockerfile
	$(call colorecho, "cat tork-indigo.sh >> tork-defaults/hooks/chroot")
	cat tork-indigo.sh >> tork-defaults/hooks/chroot

	$(call colorecho, "use casper for vivid")
	## work around for casper-rw partition http://askubuntu.com/a/725709/469233
	echo "wget http://archive.ubuntu.com/ubuntu/pool/main/c/casper/casper_1.360_amd64.deb -O /tmp/casper_1.360_amd64.deb" >> tork-defaults/hooks/chroot
	echo "sudo dpkg -x /tmp/casper_1.360_amd64.deb /tmp/casper-src" >> tork-defaults/hooks/chroot
	echo "sudo cp /tmp/casper-src/usr/share/initramfs-tools/scripts/casper /usr/share/initramfs-tools/scripts/casper" >> tork-defaults/hooks/chroot

	$(call colorecho, "cat depends.txt >> tork-defaults/depends.txt")
	cat depends.txt >> tork-defaults/depends.txt

	# build
	$(call colorecho, "build tork-defaults package")
	cd tork-defaults; dpkg-buildpackage -us -uc

	## ubuntu-defaults-image
	$(call colorecho, "run ubuntu-defaults-image (config part)")
	#cat `which ubuntu-defaults-image` | sed 's@lb config@lb config username=username hostname=ros-indigo@' | sed 's@^\(.*lb build.*\)$$@echo \1 > tork-defaults-build@'> tork-defaults-config
	cat `which ubuntu-defaults-image` | sed 's@^\(.*lb build.*\)$$@echo \1 > tork-defaults-build@'> tork-defaults-config
	chmod u+x ./tork-defaults-config
	./tork-defaults-config --package tork-defaults_0.1_all.deb --components main,restricted,universe,multiverse # --ppa xorg-edgers/ppa

	# add-apt-repository generate fiels in archvies, since lb config cleans config/archives
	$(call colorecho, "make add-apt-repository")
	make add-apt-repository
	cp archives/* ./config/archives/

	$(call colorecho, "setup rc.local")
	mkdir -p config/includes.chroot/etc/
	cp /etc/rc.local config/includes.chroot/etc/
	$(call colorecho, "Generate the SSH keys if non-existent")
	sed -i 's%^exit 0%### Generate the SSH keys if non-existent\n[ ! -f /etc/ssh/ssh_host_dsa_key ] \&\& dpkg-reconfigure openssh-server\n\nexit 0%' config/includes.chroot/etc/rc.local
	$(call colorecho, "Remove tork-defaults.list if exists")
	sed -i 's%^exit 0%### Remove tork-defaults.list if exists\n[ -f /etc/apt/sources.list.d/tork-defaults.list ] \&\& rm -f /etc/apt/sources.list.d/tork-defaults.list\n\nexit 0%' config/includes.chroot/etc/rc.local
	cat config/includes.chroot/etc/rc.local

	$(call colorecho, "change default password")
	echo "sed -i 's@blank password@password@' /usr/share/initramfs-tools/scripts/casper-bottom/25adduser" >> config/hooks/80_change_user_name.chroot
	echo "sed -i 's@U6aMy0wojraho@U6CjLveNVb5n.@' /usr/share/initramfs-tools/scripts/casper-bottom/25adduser" >> config/hooks/80_change_user_name.chroot

	$(call colorecho, "ubiquity neeed sudo") # https://github.com/k-okada/live-cd2/issues/9
	echo "sed -i 's@sh -c@sudo sh -c@' /usr/share/applications/ubiquity.desktop" >> config/hooks/90_ubiquity.desktop.sudo.chroot


	## use latest kernel
	#$(call colorecho, "use latest kernel")
	#sed -i 's@generic-lts-wily@generic-lts-xenial@' config/chroot

	$(call colorecho, "run ubuntu-defaults-image in build part")
	chmod u+x ./tork-defaults-build
	# run twice http://blog.n-z.jp/blog/2014-07-13-build-rubylive-on-vm.html
	## running twice may failed (cp: cannot stat 'chroot/boot/vmlinuz-*': No such file or directory) error
	sudo ./tork-defaults-build
	[ -e binary.hybrid.iso ] || sudo ./tork-defaults-build

	# https://lists.ubuntu.com/archives/ubuntu-devel/2014-November/038540.html
	$(call colorecho, "EFI hack")
	./make-efi-iso.sh

iso-run:
	$(call colorecho, "run indigo.tork.iso with kvm")
	kvm --cdrom indigo.tork.iso -m 1G


### test iso with packer
packer.json: packer.json.in
	# add iso filename and md5sum
	cat packer.json.in | sed 's#@ISO_MD5SUM@#$(shell md5sum indigo-tork-*.iso | cut -f1 -d\ )#' | sed 's#@ISO_URL@#file://$(CURDIR)/$(shell ls indigo-tork-*.iso)#' | tee packer.json;

packer:
	# download packer
	wget https://releases.hashicorp.com/packer/0.10.0/packer_0.10.0_linux_amd64.zip
	dpkg-query -W -f='$${Status} $${Version}\n' unzip || sudo apt-get -y install unzip
	unzip packer_0.10.0_linux_amd64.zip
	dpkg-query -W -f='$${Status} $${Version}\n' qemu-system-x86 || sudo apt-get -y install qemu-system-x86
	vagrant plugin list | grep qemu || vagrant plugin install qemu


packer_virtualbox-iso_virtualbox.box: packer.json packer-postinstall.sh
	# build box
	./packer build --force -only=virtualbox-iso packer.json

packer_qemu_virtualbox.box: packer.json packer-postinstall.sh
	# build box
	./packer build --force -only=qemu packer.json

iso-test: packer packer.json packer_qemu_virtualbox.box
	#vagrant box remove -f iso || echo "ok"
	#vagrant destroy -f default || ecoh "ok"
	#rm -f Vagrantfile
	#vagrant box add --force iso packer_virtualbox-iso_virtualbox.box
	#vagrant box list
	#vagrant init iso
	#vagrant up



clean:
	# rm old docker image
	$(call colorecho, "reomve old docker image")
	# docker ps -a | cut -d\  -f1 | xargs docker rm
	docker images | grep none &&  sudo docker rmi $$(sudo docker images | awk '/^<none>/ { print $$3 }') || echo "oK"

	# rm ubuntu-defaults-image files
	$(call colorecho, "reomve build files")
	sudo rm -fr .build auto archives binary* chroot* config livecd.* local tork-indigo.sh depends.txt docker-build

	$(call colorecho, "reomve cache")
	sudo rm -fr cache

	$(call colorecho, "reomve template files")
	# rm ubuntu-defaults-template
	sudo rm -fr tork-defaults*

help:
	@echo "docker-build: build docker image"
	@echo "docker-run:   run docker image"
	@echo "iso-build:    build iso image"
	@echo "iso-run:      run iso image in kvm"


