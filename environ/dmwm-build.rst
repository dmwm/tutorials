Software packaging and distribution in CMS
------------------------------------------

**This tutorial was reviewed on 2015-07-23. If you find problems or
need help, send to `hn-cms-webInterfaces`.** 

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
`SiteDB <https://github.com/cms-sw/cmsdist/blob/comp_gcc481/sitedb.spec>`_
but also non-CMS software like
`cherrypy <https://github.com/cms-sw/cmsdist/blob/comp_gcc481/cherrypy.spec>`_.

The tag or branch of `cmsdist` you use represents the set of packages
you will use as a configuration, including the compiler, etc. You **should
never, ever use the master HEAD of `cmsdist`**. Instead you should always
start from a concrete `cmsdist` tag or branch corresponding to a desired
or existing configuration. From there, you update individual spec files,
patches, etc, and eventually contribute your changes back.

The HTTP Group provides safe ``HGYYMM*`` `cmsdist` tags in 
a ~monthly basis. These tags are always announced in the
`webInterfaces <https://hypernews.cern.ch/HyperNews/CMS/get/webInterfaces.html>`_
forum, so you should probably subscribe there. You can search for the latest
``HGYYMM*`` tag by going to the
`cmsdist area in github <https://github.com/cms-sw/cmsdist/>`_,
clicking on the "branch: master" button, then selecting the "Tags" tab and
entering ``HG`` in the search box. The first ``HGYYMM*`` tag to appear should
be the latest.

There are two main `cmsdist` branches used for the COMP (non CMSSW) projects
that you need to be aware of:
the `comp_gcc481 <https://github.com/cms-sw/cmsdist/tree/comp_gcc481>`_
and the `comp_gcc493 <https://github.com/cms-sw/cmsdist/tree/comp_gcc493>`_.
The former is the current production used configuration, while the later
is a candidate for next production which is based on gcc493 and python2.7.

In general, when targeting the next production ``HGYYMM*`` release, you must
stick using the (HEAD of) `comp_gcc481` branch. However, if you need to
provide a critical fix to some older release, you start from the corresponding
``HGYYMM*`` tag.

Once you know which `cmsdist` configuration to use,
you need to find the corresponding `pkgtools` tag or branch. As with
`cmsdist`, you should never use the master HEAD of `pkgtools` as from
time to time changes may be introduced there which trigger a full rebuild
and/or will not work with your particular `cmsdist` tag. You should refer to
the following table to figure out which 
`pkgtools` commit/tag to use:

===================================================== ====================================================================
**cmsdist**                                           **pkgtools**              
----------------------------------------------------- --------------------------------------------------------------------
HG1609* and newer                                     use HEAD of the V00-30-XX branch
branches comp_gcc481 or comp_gcc493 up to HG1608*     on the V00-22-XX branch, stick with commit 434bf060200793b0120e0027f
HG1305* and newer on the comp branch                  on the V00-21-XX branch, stick with commit b174441c2295f1b30c5ff6494 
HG1205* to HG1304*                                    use HEAD of the V00-20-XX branch
HG120[1234]* and older tags                           not supported anymore
===================================================== ====================================================================


Build machines and the build environment
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The building process with `pkgtools` requires a few system packages
to be installed. Although we could build them all, sometimes it
is not worth the effort to maintain them. This is the case, for
instance, of ``perl``. So if you ever need to build a perl dependent
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
system-wide, it would fail.

In order to have an enough but clean environment for the builds,
we setup a few *build machines*. All the official COMP RPMs are
built and (pre)released from such machines. Their hostnames
and location of the official build logs are: ::

   vocms022:/build/dmwmbld/logs/dmwmbld: for SLC6 builds
   macms07:/build1/dmwmbld/logs/dmwmbld: for OSX builds

You should request access to the above machines too so you
can test the building of your RPMs with a correct environment
before asking them to be (pre)released. To get access to
SLC6 build machine you should write to the
`webInterfaces <https://hypernews.cern.ch/HyperNews/CMS/get/webInterfaces.html>`_
forum asking to be included in the `cms-comp-devs` e-group. For the OSX build
machine, you should ask in the
`hn-cms-sw-develtools <https://hypernews.cern.ch/HyperNews/CMS/get/sw-develtools/1849.html>`_
forum because the OSX build machines are managed by the Offline group. Also
put the `cms-http-group` in CC for the OSX request.

During the development phase, you might feel more convenient
to build your RPMs locally (i.e. on your MacOS laptop). If your local
environment is too messy or is missing some needed system packages,
the other option you have is to `setup a devvm <vm-setup.html>`_ and
do your builds from there. These devvms have a very similar environment
to the build machines.


CMS RPMs and repositories
^^^^^^^^^^^^^^^^^^^^^^^^^

The `cms repository <http://cmsrep.cern.ch/>`_ stores and serves
all the RPMs we build and release with `pkgtools`. There are three main (official)
repositories hosted there: `cmssw <http://cmsrep.cern.ch/cmssw/cms/>`_,
`comp <http://cmsrep.cern.ch/cmssw/comp/>`_ and
`comp.pre <http://cmsrep.cern.ch/cmssw/comp.pre/>`_. We don't store
all the RPMs on the same repository because CMSSW and DMWM have
different use cases, development workflows and release policies.

For DMWM builds, you'd only care about ``comp`` and ``comp.pre``. This later
hosts the pre-release RPMs, which mean RPMs that were tested but possibly
not fully validated yet. On the other hand, the ``comp`` repository shall
contain only production-ready, validated, RPMs. We also
separate them in these two repositories because every once in a while
the repository index could become broken due to problems during the upload
of RPMs, so we want to make sure the production-ready RPMs will be
available more reliably. This separation also allows us to keep ``comp``
on a clean state which makes it quicker to generate/upload urgent RPMs
during a scheduled intervention.

Currently, the comp repositories have RPMs for the following architectures:

======================================= =========== ===============
**ARCH**                                in **comp** in **comp.pre**
--------------------------------------- ----------- ---------------
slc7_amd64_gcc630 (upcoming production)      X
slc6_amd64_gcc493 (current production)       X
slc6_amd64_gcc481 (previous production)                    X
slc6_amd64_gcc461 (deprecated)               X             X
slc5_amd64_gcc461 (deprecated)               X             X
slc5_amd64_gcc434 (deprecated)               X             X
osx106_amd64_gcc461 (deprecated)             X             X
osx108_amd64_gcc481 (deprecated)                           X
======================================= =========== ===============

Besides the official repositories, there are a bunch of other "private"
repositories. Every developer gets its private snapshot of a target
repository automatically when they upload RPMs. If you build against
``comp.pre``, your private repository will be called ``comp.pre.you``, 
where ``you`` is your cern username. See for instance
`comp.pre.diego <http://cmsrep.cern.ch/cmssw/comp.pre.diego/>`_.
Similarly, if you build against ``comp``, your private repository
will be called ``comp.you``.

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

Only the COMP release managers can manually upload new RPMs to the
``comp`` and ``comp.pre`` repositories.

However, the RPMs for the ``slc6_amd64_gcc481`` and ``slc6_amd64_gcc493``
architecture are released automatically whenever changes are
committed to the branches `cmsdist/comp_gcc481` and
`cmsdist/comp_gcc493`. The COMP release managers review/test pull
requests made against those branches and would push them
upstream whenever approved. DMWM developers should therefore
send their material through pull requests there. See
`Requesting to release COMP RPMs`_ for instructions.

The requests would be taken during the ~monthly release cycles, but
major changes and requests for commissioning new services or packages
may take longer and their timelines are often discussed on the 
`DMWMReleasePlanning <https://twiki.cern.ch/twiki/bin/viewauth/CMS/DMWMReleasePlanning>`_
meetings.

If you come into a situation where none of the
release managers are responding to an urgent request, you
could deploy the RPMs directly from your private RPM repository. That is,
the RPMs you got uploaded to ``comp.pre.you`` or ``comp.you``. Provided
you used the build
machines when building them, they shall work exactly the same as RPMs
from ``comp`` or ``comp.pre``. Alternatively, you could ask anybody
with push rights to `cmsdist` to push your changes, then use the
RPMs that eventually get uploaded by the build-agent.

Since you have full control of your private, ``comp.pre.you`` and ``comp.you`` 
RPM repositories, you can upload RPMs to it at any time,
**without holding on anybody nor on a robot like the build-agent**.


Building RPMs and releasing to a private repository
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Once you know which `cmsdist` and `pkgtools` tags to use
(see `Getting pkgtools and cmsdist`_), have got access to a proper
build environment (see `Build machines and the build environment`_)
and understood what is the target RPM repository (i.e. ``comp.pre``)
to use, it is time for hands on!

The following example commands build a new SLC6 RPM for the wmagent
project. It uses the HEAD of the `cmsdist/comp_gcc481` branch for the
configuration, and the build targets the ``comp.pre`` repository.
On the SLC6 build machine: ::

  # prepare a build area
  mkdir -p /build/$USER
  cd /build/$USER
  (git clone -b V00-22-XX https://github.com/cms-sw/pkgtools.git && cd pkgtools && git reset --hard 434bf060200793b0120e0027f)
  (git clone https://github.com/cms-sw/cmsdist.git && cd cmsdist && git checkout comp_gcc481)

  vi cmsdist/wmagent.spec # do some changes to it (i.e. bump new version)

  pkgtools/cmsBuild -c cmsdist --repository comp.pre \
    -a slc6_amd64_gcc481 --builders 8 -j 5 --work-dir w \
    build wmagent-dev

  pkgtools/cmsBuild -c cmsdist --repository comp.pre \
    -a slc6_amd64_gcc481 --builders 8 -j 5 --work-dir w \
    upload wmagent-dev --upload-user=$USER

These commands will result in uploading the new RPMs to
``comp.pre.you``, **not** to ``comp.pre``! The ``--repository comp.pre``
option basically tell it to "mirror repository from comp.pre to
comp.pre.you, then upload any new produced RPMs to comp.pre.you". In
order to be able to upload anything, please first subscribe to the
`cms-comp-devs` e-group.

Note that, athough only the wmagent package (the ``wmagent.spec`` file)
was changed, we requested building/uploading everything deriving from
the ``wmagent-dev`` package. This later is a meta-package, that is, package
that does not contain any code, but only depends on other packages,
including ``wmagent``. You could, instead, have built/uploaded only
``wmagent``, but while deploying services, it is often the case where
it needs other external services or tools deployed together (i.e. rotatelogs).
The meta-package not only makes building/uploading changes for all them
together into a single process, but can also be later used when deploying
the service to automatically determine the RPM versions of all the
services/tools you need. Besides, you must upload all your new RPMs
in a single upload command.

The most common meta-package is the ``comp`` (see the ``comp.spec`` file).
You can make changes on any spec file and use it to build/upload anything
that changed or depends on something that changed. It won't rebuild anything
that has not changed (i.e. if you changed only ``sitedb``, it won't rebuild
``dbs``).

On a second example, we show how to build RPMs for the upcoming
production architecure based on gcc493 for the whole COMP software
stack (all the COMP projects). From the SLC6 build machine, do ::

  # prepare a build area
  mkdir -p /build/$USER
  cd /build/$USER
  (git clone -b V00-22-XX https://github.com/cms-sw/pkgtools.git && cd pkgtools && git reset --hard 434bf060200793b0120e0027f)
  (git clone https://github.com/cms-sw/cmsdist.git && cd cmsdist && git checkout comp_gcc493)

  vi cmsdist/sitedb.spec # do some changes to it (i.e. bump new version)

  pkgtools/cmsBuild -c cmsdist --repository comp \
    -a slc6_amd64_gcc493 --builders 8 -j 5 --work-dir w \
    build comp

  pkgtools/cmsBuild -c cmsdist --repository comp \
    -a slc6_amd64_gcc493 --builders 8 -j 5 --work-dir w \
    upload comp --upload-user=$USER

Differently from the previous example, the `cmsdist` branch here
is ``comp_gcc493``, the architecture is ``slc6_amd64_gcc493``, and the
repository is ``comp``.


Installing CMS RPMs
^^^^^^^^^^^^^^^^^^^
RPMs of projects that have `deployment scripts <https://github.com/dmwm/deployment>`_
can be installed as shown in the `devvm setup <vm-setup.html>`_ instructions.

When deploying on a non-devvm machine, you may need to install a few
bare minimum system packages. Depending on the project you are installing,
you may also need to setup system accounts, install grid CA certificates, etc.
See the
`system deploy <https://github.com/dmwm/deployment/blob/master/system/deploy>`_.
On CERN VOBoxes, this system pre-configuration is usually done in puppet.

If you want to install a raw RPM because you don't have a deployment script
for it yet, you can use the following instructions: ::

   export SCRAM_ARCH=slc6_amd64_gcc481  # or slc6_amd64_gcc493
   REPO=comp.pre # Or comp.pre.you if you are installing from your private repo
   mkdir cms-comp; cd cms-comp
   wget http://cmsrep.cern.ch/cmssw/$REPO/bootstrap.sh
   sh ./bootstrap.sh -architecture $SCRAM_ARCH -path $PWD -repository $REPO setup
   source ./$SCRAM_ARCH/external/apt/*/etc/profile.d/init.sh
   apt-get update
   apt-get -y install <RPM>


Requesting to release COMP RPMs
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Fork `cmsdist` in github, clone it from your private github account and
push your spec changes there (i.e. either to the `comp_gcc481` or the
`comp_gcc493` branches). Then send a pull request to merge them into
the correspinding `comp_gcc481 branch of cmsdist <https://github.com/cms-sw/cmsdist/tree/comp_gcc481>`_,
or into the `comp_gcc493 branch <https://github.com/cms-sw/cmsdist/tree/comp_gcc493>`_,
depending on your target architecture.

See `Creating feature branches and making a pull request <dev-git.html>`_
for detailed instructions if you are not familiar with GIT.

On the description of the pull request, please provide
a short summary of what is changing, and **tell explictly** when should
the release manager pick it up. It is often the case that
you have to hold on including some changes that affect other services
(i.e. in the service API level, not code) that are not yet ready for them.
If not specified, we'd usually include on the next upcoming release, but
we might ignore it if we judge it can disrupt anything important.

The pull requests will then be tested automatically by the build-agent, which
will post the result as a comment and change the status of the pull request
accordingly to the build result. If it fails, it usually means the changes could
not be merged or the build itself failed. You can check the build-agent log as
pointed by the test results to find out what went wrong.

If you later fix the problem, or simply update the pull request with more
commits, the build-agent should detect the changes and re-test them. You
don't need to close the pull request and open a new one. It is enough to push
your changes to the same source branch on your forked copy of the git repository
in github.

However, if in the meantime your PR has been approved by the COMP
release managers and pushed upstream, then **do not update** anymore the PR.
Instead, close it (if not yet done) and re-do the process from the beginning,
making sure your source branch contains the updated changes pushed upstream.
