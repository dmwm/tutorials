Virtual machine setup
---------------------

CERN VMM virtual machine
^^^^^^^^^^^^^^^^^^^^^^^^

Follow the `HTTP group dev-vm instructions <https://cern.ch/cms-http-group/dev-vm.html>`_.

Local virtual machine
^^^^^^^^^^^^^^^^^^^^^

These instructions create `Scientific Linux <http://scientificlinux.org>`_
5.7 virtual machine under VirtualBox 4.1.x. You can another hypervisor if
you prefer; the translation should be very straightforward. `VirtualBox
<http://www.virtualbox.org>`_ is easy to use and free for uses such as
this, so an attractive choice if you don't have another hypervisor. The
instructions are somewhat geared towards CERN-like environment. You can
adjust them to your local site conventions as far as groups, time servers,
and such.

First create a new virtual machine labelled *SL5.7*, Linux/RedHat (64-bit),
at least 2048 MB RAM. Create a new start-up disk: VDI, dynamically allocated,
40 GB in size. Download the `install boot image
<http://cern.ch/linux/scientific5/docs/repository/cern/slc5X/x86_64/images/boot.iso>`_
and save it as ``boot_sl7_x86_64.iso`` in your downloads folder. Attach it
on IDE into your VM. Set networking to *bridged* mode, and give the VM a
pre-allocated fixed MAC address.

  There are three main reasons for bridged networking and a preallocated
  MAC address. The first is that it's a great deal easier to SSH into and
  use the web server in the VM when your VM appears as any other server
  on your LAN with bridged networking. The second reason is that in order
  to get a host certificate for your VM, your site will likely require you
  to register the hostname and the MAC address. Specifically many sites,
  including CERN, will not grant a host certificate for a laptop. Third is
  that in order to get myproxy renewal rights, your host needs a stable
  name, and to get one you typically need a pre-registered MAC address.

Install minimal SL5.7 server into the VM using the boot image:

 * Language: English; Keyboard: us; Method: HTTP, DHCP no IPv6,
   ``linuxsoft.cern.ch``, ``/cern/slc5X/x86_64/``

 * OK to initialise partition table

 * Remove all partitions and create default layout:
   sda1:/boot 101 MB, sda2:LVM VG00 [LV01 swap 4000 MB, LV00 / ext3 rest]

 * Install grub loader [default]

 * Network: eth0, IPv4 DHCP, IPv6 Disabled, hostname via DHCP [default]

 * Region: Europe/Zurich, system clock uses UTC [default]

 * Set root password

 * Installation: server, customize now

   - Clear everything in: Desktop environments, Servers, Cluster Storage,
     Clustering, SLC Customizations

   - Applications: Text-based Internet (only)

   - Development: Development Libraries, Tools (only)

   - Base System: Administrative Tools, Base, Java (only)

 * After install remove CD, reboot into first boot:

   - Authentication: MD5 + shadow (no kerberos);

   - Firewall: enabled, SELinux: enforcing;
     Customize: ssh, Other ports: empty (remove afs3-callback:udp)

   - Keyboard: U.S. English

   - Network: DNS: Hostname: (give a name) (all other defaults)

   - System services: (defaults) + ntpd

   - Timezone: Europe/Zurich, system clock uses UTC; use ntp,
     servers: ip-time-{0,1,2}.cern.ch

   - Sound card: defaults (Intel 82801AA-ICH)

Now login as root and run the following, possibly adjusted for your site::

  vi /etc/ntp.conf   # server ip-time-{0,1,2}.cern.ch
  service ntpd restart
  yum -y update
  yum -y install zsh
  yum -y clean packages
  vi /etc/sudoers    # uncomment "%wheel ALL=(ALL) NOPASSWD: ALL"

  ME=<your_afs_login>
  echo your.account@cern.ch > /root/.forward
  groupadd -g 1399 zh
  useradd -M -g zh -G wheel -s /bin/zsh -u 12345 -c "Your Name" -d /home/$ME $ME
  passwd $ME
  mkdir -p /home/$ME /data
  chown -R $ME:zh /home/$ME /data

  # install guest additions
  mount /dev/cdrom /media && cd /media
  sh ./VBoxLinuxAdditions-amd64.run
  cd /; umount /dev/cdrom

  # upgrade zsh (optional)
  cp -p /bin/zsh{,.old}
  cd /tmp
  wget http://downloads.sourceforge.net/zsh/zsh-4.3.12.tar.bz2
  tar jxf zsh-*.tar.bz2
  cd zsh-*/
  ./configure --prefix=/usr --libdir=/usr/lib64 zsh_cv_sys_link=no
  make -j 2
  make install # DESTDIR=/tmp/foobar for test
  rm -f /bin/zsh; ln /usr/bin/zsh /bin/zsh
  rm -fr /tmp/zsh*

  # turn off
  shutdown -h 0

Create VM snapshot for installed state. Restart. Run post-install, e.g.
copy your shell environment::

  scp ~/.z{log{in,out},sh{env,rc}} your-vm-host:
  scp -rp ~/.globus your-vm-host:

Your VM is ready for use. SSH into it and deploy servers normally as per
`dev-vm instructions <https://cern.ch/cms-http-group/dev-vm.html>`_::

  # one-time preparation
  mkdir -p /tmp/foo
  cd /tmp/foo
  svn co svn+ssh://svn.cern.ch/reps/CMSDMWM/Infrastructure/trunk/Deployment cfg
  sudo -l
  cfg/Deploy -t dummy -s post $PWD system/devvm
  rm -fr /tmp/foo

  sudo yum -y install voms-clients myproxy
  B=/afs/cern.ch/project/gd/LCG-share/3.2.8-0
  sudo scp -rp you@lxplus.cern.ch:$B/glite/etc/vomses /etc/vomses
  sudo scp -rp you@lxplus.cern.ch:$B/external/etc/grid-security/vomsdir /etc/grid-security

  # server installation, using admin tools as shortcuts
  cd /data
  rsync -avu cmsweb@lxplus.cern.ch:private/conf/ /data/auth/
  chgrp -R _sw /data/auth
  chmod ug=r,o-rwx $(find /data/auth -type f)
  chmod u=rwx,g=rx,o-rwx $(find /data/auth -type d)

  A=/data/cfg/admin REPO="-r comp=comp.pre" VER=1111d
  PKGS="admin frontend base couchdb das dbs dbsweb dqmgui filemover mongodb phedex"
  PKGS="$PKGS overview sitedb/legacy stagemanager t0datasvc t0mon reqmgr workqueue"
  $A/InstallDev -s image -v hg$VER -a $PWD/auth ${=REPO} -p "$PKGS"
  $A/InstallDev -s start
  $A/InstallDev -s status

  # cleanup
  cd /data
  $A/InstallDev -s stop
  crontab -r
  killall python
  sudo rm -fr [^aceu]* .??* current enabled

Environment on a Mac OS X system
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This is really not a virtual machine environment, but there is experimental
support for settings this up on an OS X laptop. This has only been tested
with Snow Leopard::

  # Fake enough of grid environment
  sudo mkdir -p /etc/grid-security
  B=/afs/cern.ch/project/gd/LCG-share/3.2.8-0
  GS=/etc/grid-security BGS=$B/external/etc/grid-security
  sudo rsync -av --delete you@lxplus.cern.ch:$B/../certificates $GS/certificates/
  sudo rsync -av --delete you@lxplus.cern.ch:$B/glite/etc/vomses/ /etc/vomses/
  sudo rsync -av --delete you@lxplus.cern.ch:$B/glite/etc/vomses/ /etc/vomses/
  sudo rsync -av --delete you@lxplus.cern.ch:$BGS/vomsdir/ $GS/vomsdir/
  sudo chown -R root:$(id -gn root) /etc/grid-security /etc/vomses

  # Create accounts and all the rest; this installs into /users/cmssw/test
  # instead of using /data. You may need to iterate and copy a host cert
  # from somewhere into machine if the default rule doesn't work.
  mkdir -p /tmp/foo
  cd /tmp/foo
  svn co svn+ssh://svn.cern.ch/reps/CMSDMWM/Infrastructure/trunk/Deployment cfg
  sudo -l
  CMS_DEV_ROOT=/users/cmssw/test cfg/Deploy -t dummy -s post $PWD system/devmac
  cd; rm -fr /tmp/foo

  # Install software using roughly standard dev-vm instructions.
  cd /users/cmssw/test
  rsync -avu cmsweb@lxplus.cern.ch:private/conf/ $PWD/auth/
  chgrp -R _sw $PWD/auth
  chmod ug=r,o-rwx $(find $PWD/auth -type f)
  chmod u=rwx,g=rx,o-rwx $(find $PWD/auth -type d)

  cd /users/cmssw/test
  A=$PWD/cfg/admin REPO="-r comp=comp.pre" VER=1111a
  PKGS="admin frontend base couchdb das dbs dbsweb dqmgui filemover mongodb phedex"
  PKGS="$PKGS overview sitedb/legacy stagemanager t0datasvc t0mon reqmgr workqueue"
  $A/InstallDev -s image -v hg$VER -a $PWD/auth ${=REPO} -p "$PKGS"
  $A/InstallDev -s start
