Software packaging and distribution in CMS
------------------------------------------

CMS uses its own build facility to pack its software into RPMs and to
distribute them. It is used by both the Offline group to distribute CMSSW
and its related tools (i.e. the Tag Collector), and the Computing group to
distribute most of its software (both the DMWM and non-DMWM projects).


Getting pkgtools and cmsdist
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The CMS build tool is called `pkgtools <https://github.com/cms-sw/pkgtools>`_,
and the configuration set used as input for it to generate RPMs is called
`cmsdist <https://github.com/cms-sw/cmsdist>`_.

`cmsdist` contains build recipes for many software
packages (both CMS and non-CMS) in the form of rpm spec files, as well
as patches and some files for the build infrastructure. Take a look into
its files. There you can find the spec files to build
`SiteDB <https://github.com/cms-sw/cmsdist/blob/master/sitedb.spec>`_
but also non-CMS software like
`cherrypy <https://github.com/cms-sw/cmsdist/blob/master/cherrypy.spec>`_.

The tag or branch of `cmsdist` you use represents the set of packages
you will use as a configuration, including the compiler, etc. You should
never, ever use the master HEAD of `cmsdist`. Instead you should always
start from a concrete `cmsdist` tag or branch corresponding to a desired
or existing configuration. From there, you update individual spec files,
patches, etc, and eventually contribute your changes back.


The HTTP Group provides safe ``HGYYMM*`` `cmsdist` tags in 
a ~monthly basis. These tags are always announced in the
`webInterfaces <https://hypernews.cern.ch/HyperNews/CMS/get/webInterfaces.html>`_
forum, so you should probably subscribe there. You can search for the latest
``HGYYMM*`` tag by going to the
`cmsdist area in github <https://github.com/cms-sw/cmsdist/>`_,
clicking on the "banch: master" button, then selecting the "Tags" tab and
entering ``HG`` in the search box. The first ``HGYYMM*`` tag to appear should
be the latest. The
`cmsdist comp <https://github.com/cms-sw/cmsdist/tree/comp>`_
branch is also kept up-to-date with the latest release candidate changes.
In general, if you targeting the next production ``HGYYMM*`` release, you must
stick using the `comp`
branch. Otherwise, if you need to provide a critical fix to some old release,
you start from the corresponding ``HGYYMM*`` tag. If you have no idea of the last
workable tag used by you project, you could check the RPM
`releases <https://twiki.cern.ch/twiki/bin/view/CMS/DMWMBuildsStatusReleases>`_
and `pre-releases <https://twiki.cern.ch/twiki/bin/view/CMS/DMWMBuildsStatusPreReleases>`_
pages to find out when the last RPM was built and the tag that was used.

Once you know which `cmsdist` configuration to use,
you need to find the corresponding `pkgtools` tag or branch. As with
`cmsdist`, you should never use the master HEAD of `pkgtools` as from
time to time changes may be introduced there which trigger a full rebuild
and/or will not work with your particular `cmsdist` tag. In general,
you'd use the HEAD of one of the
`pkgtools branches <https://github.com/cms-sw/pkgtools/branches>`_
according to the following table:

================================================= =========================
**cmsdist**                                       **pkgtools**
------------------------------------------------- -------------------------
anything from the comp_gcc481 branch              use the V00-22-XX branch
HG1305* and newer, or the HEAD of the comp branch use the V00-21-XX branch   
HG1205* to HG1304*                                use the V00-20-XX branch
HG120[1234]* and older tags                       not supported anymore
================================================= =========================


Build machines and the build environment
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The building process with `pkgtools` requires a few system packages
to be installed. Although we could build them all, sometimes it
is not worth the effort to maintain them. This is the case, for
instance, of perl. So if you ever need to build a perl dependent
package, you must have it system wide installed.

On another point of view, when we specify that a package has a
dependency on some other package, we want to make sure our build
will stick using this other package we build together. We want
it to fail if not all the symbols can get solved with
the dependency package we are providing. When you have plenty
of system-wide packages installed, it often happens you get
fooled by the fact the missing symbols are found on the system
packages. Then when you try to deploy the resulting RPM on a
different machine where the missing symbols are not available
system-wide, you'd fail.

In order to have an enough but clean environment for the builds,
we setup a few *build machines*. All the official DMWM RPMs are
built and (pre)released from such machines. Their hostnames
and location of the official build logs are: ::

   vocms106:/build/dmwmbld/logs/dmwmbld: for SLC5 builds
   vocms178:/build/dmwmbld/logs/dmwmbld: for SLC6 builds
   macms07:/build1/dmwmbld/logs/dmwmbld: for OSX builds

You can (should) request access to these machines too so you
can test the building of your RPMs with a correct environment
before asking them to be (pre)released. To get access to
SLC5 and SLC6 build machines you should write to the
`webInterfaces <https://hypernews.cern.ch/HyperNews/CMS/get/webInterfaces.html>`_
forum asking to be included in the `cms-comp-devs` e-group. For the OSX build
machine, you should ask in the
`hn-cms-sw-develtools <https://hypernews.cern.ch/HyperNews/CMS/get/sw-develtools/1849.html>`_
forum because the OSX build machines are managed by the Offline group. Also
put the `cms-http-group` in CC for the OSX request.

During the development phase, you might find quicker or more convenient
to build your RPMs locally, even on your MacOS laptop! If your local
environment is too messy or is missing some needed system packages,
the other option you have is to `setup a devvm <vm-setup.html>`_ and
do your builds from there. These devvms have a very similar environment
to the build machines.


CMS RPMs and repositories
^^^^^^^^^^^^^^^^^^^^^^^^^

The `cms repository <http://cmsrep.cern.ch/>`_ stores and serves
all the RPMs we build with `pkgtools`. There are three main (official)
repositories hosted there: `cmssw <http://cmsrep.cern.ch/cmssw/cms/>`_,
`comp <http://cmsrep.cern.ch/cmssw/comp/>`_ and
`comp.pre <http://cmsrep.cern.ch/cmssw/comp.pre/>`_. We don't store
all the RPMs on the same repository because CMSSW and DMWM have very
different use cases, development workflows and release policies.

For DMWM builds, you only care about ``comp`` and ``comp.pre``. This later
hosts the pre-release RPMs, which mean RPMs that were not validated yet.
The ``comp`` repository shall contain only production-ready RPMs. We also
separate them in these two repositories because every once in a while
the repository index can get broken due to problems during the upload
of RPMs, so we want to make sure the production-ready RPMs will be
available more reliably. This separation also allows us to keep ``comp``
on a clean state which makes it quicker to generate/upload urgent RPMs
during a scheduled intervention.

Currently, the comp repositories have RPMs for the following architectures:

======================================= =========== ===============
**ARCH**                                in **comp** in **comp.pre**
--------------------------------------- ----------- ---------------
slc5_amd64_gcc434 (deprecated)               X             X
slc5_amd64_gcc461 (current production)       X             X
slc6_amd64_gcc461 (testing)                  X             X
slc6_amd64_gcc481 (upcoming production)                    X
osx106_amd64_gcc461 (testing)                X             X
osx108_amd64_gcc481 (testing)                              X
======================================= =========== ===============

Besides the official repositories, there are a bunch of other "private"
repositories. Every developer gets its private snapshot of a target
repository automatically when they upload RPMs. If you build against
``comp.pre``, your private repository will be called ``comp.pre.you``, 
where ``you`` is your cern username. See for instance
`comp.pre.diego <http://cmsrep.cern.ch/cmssw/comp.pre.diego/>`_.

Please note that none of these RPM repositories are compatible
with yum repositories. **CMS RPMs are not yum RPMs!**. They don't
interoperate directly, so you can't tell in your CMS RPM to depend
on a yum RPM. They are incompatible for various reasons:

- yum RPMs require superuser access to get installed because its 
  database and index is stored system-wide. This is a show-stopper
  for distributing our CMS software on various places where packages
  must be installed in the user space (i.e. on lxplus);
- using yum would require it to be instaled everywhere. This is
  clearly not the case for MacOSX and also newer ARM-based systems.
  The CMS RPMs don't require any particular technology and is
  therefore pretty much flexible to target the various different
  platforms;
- the CMS RPMs allows you to install different versions of the same
  package at the same time. This is very painful, yet impossible in
  many cases, to be achieved with yum. We often need to have different
  gcc, openssl and python versions installed at the same time. The CMS
  tools isolate the dependency environment appropriatedly so that
  the dependency chain used by pkg A don't stomp the pkg B dependency
  chain. It is often the case that all but a single application can't
  yet use the newer version of a common dependency like openssl;
- in CMS RPMs, we need to prune more aggressively the content of the
  RPMs so that we can keep the overall size of the installed software
  into some reasonable enough size to transfer it quicker. In particular,
  we delete doc files, static libraries and disable package features
  that are not used anywhere in CMS but just bloat the size of a package.
  Doing this kind of cleaning for yum RPM repositories is impossible
  as their official repositories must keep docs and other package
  features to match the various other use cases. Even if we run
  our own yum repository, it may be tricky to guarantee the base RPMs
  get installed from our repo instead of other official ones;
- yum and other official RPM repositories have its own RPM release policies
  that on various cases don't match the CMS workflow. In particular, one
  needs to wait for a day to get a new RPM to appear in
  `Linuxsoft <http://linuxsoft.cern.ch/>`_, the main Scientific Linux
  yum repository used by VOBoxes at CERN. To avoid such policies,
  we'd need to run our own yum repository and instruct machines all
  around the world to use it. We'd them be limited to whatever the
  yum repository tools allow us to do and therefore it wouldn't
  be easily possible to define our own repository structure needed
  to catch the use cases shown on the other items above.


The CMS RPM release policy
^^^^^^^^^^^^^^^^^^^^^^^^^^

Only the COMP release managers
can upload new RPMs to ``comp``. These RPMs are essentially generated
from the same configuration used for a RPM in ``comp.pre`` that has
been fully validated in a testbed environment. These requests
are guaranteed to be taken by the COMP release managers during
working hours. If you come into a situation where none of the
release managers are responding to an urgent request, you
could deploy the corresponding (validated) RPM from ``comp.pre``.
Releasing a RPM into ``comp.pre`` can be done automatically through the
build-agent. We will see later how to request releases to both ``comp``
and to ``comp.pre``.
In particular, **you don't need to wait for
anybody's approval to deploy a production-ready RPM**.

You have full control of your private repositories (i.e. ``comp.pre.you``).
You can upload RPMs to it at any time, without waiting for anybody nor
for the build-agent. Note, however, you must upload all your new RPMs
at once because ``comp.pre.you`` will be reset automatically to latest
``comp.pre`` snapshot just before the RPM gets uploaded. In particular,
you can't upload pkg X and then pkg Y. You should instead put X and Y
on the same upload command request. On some cases, you may find more
convenient to create a dummy package Z that does nothing but only depends
on both X and Y, then always build/upload Z instead. You can upload
RPMs to ``comp.pre.you`` from anywhere. However, you must subscribe to the
`cms-comp-devs` e-group in order to have read access to ``comp.pre`` and
write access to ``comp.pre.you`` in cmsrep.cern.ch.


Building RPMs and releasing to a private repository
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Once you know which `cmsdist` and `pkgtools` tags to use
(see `Getting pkgtools and cmsdist`_), have got access to a proper
build environment (see `Build machines and the build environment`_)
and understood what is the target RPM repository (i.e. ``comp.pre``)
to use, it is time for hands on!

The following example commands build a new RPM for the ReqMgr project
from `cmsdist` HG1310f with target repository ``comp.pre``. ::

  # prepare a build area
  mkdir -p /build/$USER
  cd /build/$USER
  git clone -b V00-21-XX https://github.com/cms-sw/pkgtools.git
  (git clone https://github.com/cms-sw/cmsdist.git && cd cmsdist && git reset --hard HG1310f)

  vi cmsdist/reqmgr.spec # do some changes to it (i.e. bump new version)

  pkgtools/cmsBuild -c cmsdist --repository comp.pre \
    -a slc5_amd64_gcc461 --builders 8 -j 5 --work-dir w \
    build reqmgr

  pkgtools/cmsBuild -c cmsdist --repository comp.pre \
    -a slc5_amd64_gcc461 --builders 8 -j 5 --work-dir w \
    --upload-user=$USER upload reqmgr

These commands will result in uploading the new reqmgr RPM to
``comp.pre.you``, **not** to ``comp.pre``! The ``--repository comp.pre``
option basically tell it to "mirror repository from comp.pre to
comp.pre.you, then upload any new produced RPMs to comp.pre.you".


Installing CMS RPMs
^^^^^^^^^^^^^^^^^^^
RPMs of projects that have `deployment scripts <https://github.com/dmwm/deployment>`_
can be installed as shown in the `devvm setup <vm-setup.html>`_ instructions.

When deploying on a non-devvm machine, you may need to install a few
bare minimum system packages. Depending on the project you are installing,
you may also need to setup system accounts, install grid CA certificates, etc.
See the
`system deploy <https://github.com/dmwm/deployment/blob/master/system/deploy>`_.
On CERN VOBoxes, this system pre-configuration is usually done in quattor
templates.

If you want to install a raw RPM because you don't have a deployment script
for it yet, you can use the following instructions: ::

   export SCRAM_ARCH=slc5_amd64_gcc461
   REPO=comp # NOTE: use REPO=comp.pre if you are deploying from Pre-Releases
   mkdir cms-comp; cd cms-comp
   wget http://cmsrep.cern.ch/cmssw/$REPO/bootstrap.sh
   sh ./bootstrap.sh -architecture $SCRAM_ARCH -path $PWD -repository $REPO setup
   source ./$SCRAM_ARCH/external/apt/*/etc/profile.d/init.sh
   apt-get update
   apt-get -y install <RPM>


Releasing RPMs to ``comp.pre`` and ``comp``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**ATTENTION**: since the migration of cmsdist from CVS to GIT, the
build-agent is disabled, so your release requests will be handled
manually by the COMP release managers on the best effort.

Fork `cmsdist` in github, clone it from your private github account and
push your spec changes there. Then send a pull request to merge them into
the `comp branch of cmsdist <https://github.com/cms-sw/cmsdist/tree/comp>`_,
or into the `comp_gcc481 branch <https://github.com/cms-sw/cmsdist/tree/comp_gcc481>`_.
See `Creating feature branches and making a pull request <dev-git.html>`_
for detailed instructions if you are not familiar with GIT.

On the subject of the pull request, you should specify:

- the target repository: ``comp.pre`` or ``comp``;
- the architecture as used in the cmsBuild commands: ``slc5_amd64_gcc461``,
  ``slc6_amd64_gcc461``, ``osx106_amd64_gcc461``, ``slc6_amd64_gcc481``,
  ``osx108_amd64_gcc481`` or ``*`` to build in all architectures;
- the package name: i.e. ``reqmgr``.

For example: ``comp.pre slc5_amd64_gcc461 reqmgr``.

The pull requests will be handled by the build-agent at every 10 minutes. 
For requests to ``comp.pre``, the RPMs get uploaded automatically when
the build succeeds. In these cases, it also tags `cmsdist` and updates the
`Pre-Releases status page <https://twiki.cern.ch/twiki/bin/view/CMS/DMWMBuildsStatusPreReleases>`_.
The pull request is then updated with the result and closed automatically.
If you want to trigger a rebuild attempt, it is enough to reopen the pull request.

The requests to ``comp`` get built similarly, but they are **not** uploaded
automatically. The release managers will review them first. Once approved,
they get automatically uploaded and tagged, and the
`Releases status page <https://twiki.cern.ch/twiki/bin/view/CMS/DMWMBuildsStatusPreReleases>`_
also gets updated.

We always tag the `cmsdist` configuration used to upload a new RPM so that
we can keep track of how it was originated, replicate it later if needed,
and for debugging in case of problems. Note we will cleanup
automatically these tags if they are older than 3 months. However, we will
not clean an old tag if it was used to upload the last package RPM. Like
that we can ensure we can always rebuild the last RPM of a package.
