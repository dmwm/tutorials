Writing a deploy script
-----------------------

Overview
^^^^^^^^

This page describes how to maintain DM/WM software deployment recipes. All the
code resides in DM/WM github `deployment <https://github.com/dmwm/deployment>`_.
For each service there is a sub-directory which contains the deployment script
``deploy``, server :doc:`management script <ops-manage>`,
:doc:`monitoring specs <ops-monitor>` and any configuration files except
security-sensitive authentication data.

(Not) making changes to the script
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Please do not commit any changes directly. Please submit tested changes as
github pull requests as per :doc:`../environ/dev-git` guidelines.
All changes pertaining to the service -- deploy
scripts, :doc:`manage scripts <ops-manage>`, :doc:`monitoring specs <ops-monitor>`
and server configuration files -- should be supplied in the pull request. We
will review your patches, and may request changes. Once they are
approved, we will commit them to github for you. You should assume progressive
clean-up of your deployment action can happen on every new server version
submission.

How the deployment script is used
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The deployment script is used for both first installations and upgrades.
The package deployment routine should work for both situations, including
when there is nothing new to do, and when upgrade requires clean-up. The
service must be fully functional but not yet started at the end of the
installation. Commands for major one time migrations should be highlighted
as such with ``$MIGRATION`` (see below for complete details).

The script must operate on fully relocatable destination. Assume as little
as possible about the destination machine. If you have special requirements
you want to add to the script, please enquire on the
`hypernews forum <https://hypernews.cern.ch/HyperNews/CMS/get/webInterfaces.html>`_.

All application servers run under a separate daemon account. Software and
configuration is owned by yet another separate account. We encourage developers
to :doc:`use identical setup <../environ/vm-setup>`. You can try make everything work for
your app when using just a single account, but it's not something we support
centrally. To hide most of the details there are several helper functions and
the install is divided into three parts: *prep*, *sw* and *post*.

In production, different deployment stages are executed on different hosts.
Software installation, *prep* and *sw* stages, are run on the image server,
which only hosts a disk image of the software, but doesn't ever run any
applications. The entire installation is pushed from the image server to the
actual servers, currently using ``rsync`` but in future this may be a disk
image mount instead. Finally the *post* stage is executed on each server.
Thus, all host-specific configuration must be done in the *post* action; it
makes no sense for other deploy actions to refer for example to ``$host``,
it will not be the host where the servers run.

What the script should and should not do
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The deployment action should do everything required to install or upgrade
the service, bringing it fully to runnable state. The script should not stop
or start the server, that will happen outside using the
:doc:`management <ops-manage>` scripts during upgrades.

In general, deploying the server should install both the software and the
specific configuration required for CMSWEB cluster systems: production,
pre-production and development clusters. If at all possible the script should
work for any user on any system and any path. Please follow these conventions:

* The installation must be completely confined to the installation root
  (referred to as *$root* here). This implies that making any reference anywhere
  to AFS is forbidden. Explicit references to the production installation
  location */data* are also unwelcome.

* If your server software provides ability to download files from local file
  system, including CherryPy ``static_file()`` method, you must provide positive
  proof the file access so granted is guaranteed to be safe and will restrict
  itself to specific set of files. Unrestricted access to the local file system
  is prohibited, even if it's just the directory trees from your RPM-installed
  software. Exposing more sensitive directories will likely lead to immediate
  deactivation of your application.

* Instance-specific configuration files and management scripts should be
  included in the same directory as the deployment script. The files are
  usually copied automatically to ``$root/current/config/<service>``. In
  production your application should be able to read but not write the files.

* RPMs should install into ``$root/current/sw``. RPMs should be installed
  only from the comp repository and its derivatives like comp.pre.

* The software should take as command line options paths to the
  instance-specific configuration files, which should specify everything about
  the server identity. *Modifying RPM-installed software area to configure or
  patch the server software is not accepted except for old unmaintained legacy
  servers*.

* Server logs should be written to ``$root/logs/<service>`` via the
  ``rotatelogs`` program.

* Server run time state, if any, should be kept in ``$root/state/<service>``.

* Security sensitive configuration files can be found in
  ``$root/current/auth/<service>``. **These files may not be copied anywhere
  else.** You may of course refer to the path of these files in your
  configuration files.

* If your service requires a grid proxy, you need to inform us exactly how
  your service is making use of it. Use ``mkproxy`` to register the use as
  shown below. The proxy appears in ``$root/state/<service>/proxy/proxy.cert``,
  you should point $X509_USER_PROXY to this path in your server startup
  scripts.

* The service must run under application-specific daemon account. The
  configuration for this is explained below.

Typical deployment
^^^^^^^^^^^^^^^^^^

Normally, when services runs under separate accounts, the deployment is
roughly like this, where 1207c is the version of the release series
you want to install for, and the 12.07c is the corresponding deployment
tag in github: ::

    cd /data
    (git clone git://github.com/dmwm/deployment.git cfg && cd cfg && git reset --hard 12.07c)

    $PWD/cfg/Deploy -t 1207c -a -s prep /data myapp
    sudo -H -u _sw bashs -lc "$PWD/cfg/Deploy -R cmsweb@1207c -t 1207c -a -s sw /data myapp"
    $PWD/cfg/Deploy -t 1207c -a -s post /data myapp


Maintaining a deployment action
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Each service has a subdirectory parallel to the *Deploy* script. Any
sub-directories which contain a ``deploy`` script are installable products.
These scripts should contain ``deploy_<product>_<stage>`` functions as
described below. Unnecessary functions do not need to be defined. For
particularly complex deployment, the file can define other functions, as
long as the names don't conflict with those used in *Deploy* script itself
or those in other products.

The functions should have a desired target state and perform actions which
bring the system to that target. All actions should be executed always, such
that they are idempotent when there is nothing to change. This allows the
script to be used for upgrades and to fix broken installations. The script
should also handle migrations whenever service setup changes, including
cleaning up old files and preserving any precious files.

All commands always run with ``set -e`` in effect, so every command must
succeed with exit code $? equal to 0. This means that ``if`` statements must
always containing both ``then`` and ``else`` parts. It is sometimes easier to
write choices as ``case .. esac`` because of this. On the other hand you can
make "assertions" by simply executing a command which returns true for the
arguments.

The scripts enforce standard naming of directories and accounts. The names are
all determined from the sub-directory name. For
service *myapp* the names will be:

* deployment functions ``deploy_myapp_stage`` with any dashes converted to
  underscores
* configuration files under ``$root/current/config/myapp``
* server logs under ``$root/logs/myapp``
* server state under ``$root/state/myapp``
* proxy under ``$root/state/myapp/proxy/proxy.cert``
* server account ``_myapp``.

The following variables are available in all deployment actions:

* ``$variant`` is user-selected installation variant, ``default`` by default
* ``$project`` is the name of the service/project being installed
* ``$project_config_src`` is the project configuration directory fetched from github, e.g.
  ``$PWD/cfg/myapp``
* ``$project_config`` is the project configuration directory, e.g.
  ``$root/current/config/myapp``
* ``$project_auth`` is the project authentication directory, e.g.
  ``$root/current/auth/myapp``
* ``$project_state`` is the project state directory, e.g.
  ``$root/state/myapp``
* ``$project_logs`` is the project log directory, e.g. ``$root/logs/myapp``.

Describing variants
^^^^^^^^^^^^^^^^^^^

The service may optionally list names of available installation variants. ::

    deploy_myapp_variants="default offsite"

Please use this possibility sparingly. Most packages do not need this
definition. Unless user requests a specific variant, the *Deploy* script will
attempt to install the ``default`` variant. If you do not include ``default``
in the variants list, the user is always forced to make a choice when
installing the service.

Actual variant handling is done in subsequent stages. At each function call
the ``$variant`` variable will contain the user's selection. Use for example
a ``case $variant in foo ) ... ;; esac`` statement to perform commands
depending on variant choice.

Installing dependencies
^^^^^^^^^^^^^^^^^^^^^^^

The first deployment function to provide is ``deploy_myapp_deps``. ::

    deploy_myapp_deps()
    {
      deploy backend
    }

The ``deploy_myapp_deps`` should install any other services which must
be co-hosted with your service. It is always executed first when
deploying *myapp*, and ensures required dependencies are installed before your
package. For most services this should be just ``deploy backend``, but if your
service requires X509 proxy certificate, say ``deploy admin``. If your
application uses the WMCore's security module, add dependency on ``wmcore-auth``.
In some cases you may not want to have any dependencies at all, in which case
you can omit this function entirely.

Preparing directories for service installation
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The *prep* stage should normally look like this. ::

    deploy_myapp_prep()
    {
      mkproj
    }

The ``deploy_myapp_prep`` should create server working directory using
``mkproj`` function. This automatically creates project state directory
``$root/state/myapp`` and log directory ``$root/logs/myapp``; you can
suppress creating these using ``-s`` and ``-l`` options, respectively.

You can request ``mkproj`` to create additional directories. Relative
paths are relative to the project subdirectory. Absolute paths can also
be given, but they need to subdirectories of ``$root`` to keep installation
relocatable.

Use ``setgroups`` to assign correct ownership on the remaining extra
directories. The command is no-op when multiple accounts are disabled. The
first parameter is ``chmod`` argument, the second is ``chgrp`` argument and
the remaining arguments are directories. You can also give ``-R`` option if
you want to perform recursive changes, but be careful with these -- the
install actions cannot modify the group ownership on files created by the
server. The ``chmod`` argument should always be *relative*, not an absolute
mode like 775.

In production, with per-service accounts, ``mkproj`` automatically assigns
``_sw`` group to the project configuration directory so software installation
can later modify it. The *myapp* server will run under the ``_myapp`` account
and ``_myapp`` group, and needs to be able to write to the ``logs`` directory,
so the log directory is assigned ``_myapp`` group and made group-writable.

If your application needs a X509 proxy certificate, add ``mkproxy`` call
something like the following. It will automatically create a ``proxy``
subdirectory in ``$root/state/myapp/proxy``. ::

    deploy_myapp_prep()
    {
      mkproj
      mkproxy
    }

In addition to the standard commands above, any migration from version to
another are best implemented in the *prep* stage. Delete any old directories
or files here, especially if they will be on the way of *sw* stage. Prefix
such commands with the word ``$MIGRATION``.

After ``mkproj``, all commands execute with current directory in
``$root/state/myapp`` if it exists, in ``$root`` otherwise. This applies
to all subsequent installation stages, not just the *prep* one.

Installing software
^^^^^^^^^^^^^^^^^^^

The next stage is software installation: ::

    deploy_myapp_sw()
    {
      deploy_pkg comp cms+myapp
    }

In production the ``deploy_myapp_sw`` runs under ``_sw`` account and leaves
files owned by the ``_config`` group. This is to protect them so that the
running server can read the files, but not modify them. You normally just
run ``deploy_pkg`` function, which :ref:`is documented below <deploy_pkg>`.

Here we install CMS RPM package *myapp* from *comp* repository into
``$root/current/sw`` base path. The version of the package is not normally
defined in the deploy script, it is normally automatically determined
from the release series meta-package command-line -R option. In other words,
you tell the system to install "whatever is current for this release series."
Assuming this version is ``x.y.z``, we also create symlink
``$root/current/apps/myapp`` which points to
``$root/current/sw/slc5_amd64_gcc434/cms/myapp/x.y.z``. Other files such as
:doc:`management scripts <ops-manage>` and configuration files are copied
from the project configuration directory fetched from github
($project_config_src) into ``$root/current/config/myapp``
($project_config). Hence the files to be installed are determined by what
was fetched from github in the first place. (Replace *comp* with *cms* if
the software is in *cms* repository.)

If your configuration files are not fully relocatable, you may need to fix
them up with a command such as this: ::

      perl -p -i -e "s{/data}{$root}g" $project_config/myconfig.file

Post-installation actions
^^^^^^^^^^^^^^^^^^^^^^^^^

The last stage is to run post-install actions. ::

    deploy_myapp_post()
    {
      case $host in vocms53 ) disable ;; * ) enable ;; esac
      (mkcrontab; sysboot; echo "17 2 * * * $project_config/daily") | crontab -
    }

The ``deploy_myapp_post`` should record whether the service is enabled or
disabled on this particular host. Recall that in production the *prep* and
*sw* stages run on the image server, so the *post* stage needs to record
which services are actually going to be used on which hosts.

The ``deploy_myapp_post`` should also install or upgrade cron jobs for
automatic server management. The ``mkcrontab`` is just a shortcut for
current crontab minus anything which mentions ``$project_config``. To this
you should add a cron ``@reboot`` stanza to start the server automatically
on reboot, the ``sysboot`` automates this. This will invoke the
:doc:`manage script <ops-manage>` with ``sysboot`` action, which is like
``start`` but protects against spurious restarts caused by ``crond``
restarts outside system boot.

Typical other management tasks would include for example daily purging of
any old state files. The ``admin`` package installs a log archiver which
automatically compresses and stashes away all old log files into zip files,
by application and server.

Example
^^^^^^^

In general there should be little other content in your deployment action.
If your *myapp* requires no content for some of the above functions, just
leave the function out.

The complete set of functions for a standard installation with full management
automation for *myapp* would look like this. ::

    deploy_myapp_deps()
    {
      deploy backend
    }

    deploy_myapp_prep()
    {
      mkproj
    }

    deploy_myapp_sw()
    {
      deploy_pkg comp cms+myapp
    }

    deploy_myapp_post()
    {
      (mkcrontab; sysboot; echo "17 2 * * * $project_config/daily") | crontab -
    }

Documentation on internal helper commands
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

mkproj
^^^^^^

The ``mkproj`` function accepts the following options:

* ``-l`` suppresses creation of log directory.

* ``-s`` suppresses creation of state directory.

Any other arguments are directories to be created in project state directory
(if relative), or anywhere else (if absolute). The log, state directory and
any additional directories are made owned and writeable by group ``_myapp``,
but just the directory itself, not its contents. The project configuration
directory is created owned and writeable by group ``_sw``.

setgroup
^^^^^^^^

The ``setgroup`` function accepts any options which are valid to ``chmod`` or
``chgrp``, for example ``-R`` to apply the operation recursively. Non-option
arguments are mode, group and possibly empty list of path names to apply the
operation on. It the list is non-empty, setgroup first applies the ``chgrp``
operation with the requested group, then ``chmod``.

.. _deploy_pkg:

deploy_pkg
^^^^^^^^^^

The ``deploy_pkg`` function accepts the following options:

* ``-a dest[:source]`` adds the pair to list of authentication files to
  install under ``$root/current/auth``. Note that if you use this argument,
  you need to define ``deploy_myapp_auth`` function as described below.

* ``-l name`` creates symbolic link *name* to the installed package under
  ``$root/current/apps``. By default the link is named the same as the
  installed RPM, for example *myapp* when installing RPM ``cms+myapp+1.2.3``.

The remaining arguments are either none, and two or three arguments:

* Repository, usually ``comp``.

* Package to install, usually ``cms+app``.

* Optionally an explicit package version to install. Normally this is omitted,
  which is interpreted as 'auto', which means the version is determined from
  the meta-package given with -R command line option. The version specified
  here can be overridden from command line.

The function first installs the RPM if any was provided. A link to the
extracted RPM package is automatically created under ``$root/current/apps``.
If the RPM was omitted, the other parts below are still done. This can be
useful to install for example just authentication file or scripts for the
configuration area without RPMs.

Next the function copies any files from ``$project_config_src`` into
``$project_config``, i.e. the github repository directory contents to
``$root/current/config/myapp``. Everything in the directory is assigned to
``_config`` group, readable but not writeable by the group.

Finally the function installs any authentication files specified with ``-a``
option. These can be simple file names such as ``phedex/DBParam``, or source /
destination name pair such as ``myapp/foo:myapp/bar``. You'd use the latter to
select the file dynamically by some logic, such as using different files for
production, pre-production and development.

The authentication files are searched under the directory given with ``-p``
command line option to *Deploy*. If no such file exists, such as when no
``-p`` option is used at all, the ``deploy_myapp_auth`` function will be
invoked with the destination file name as argument. The function should output
a template auth file, with actual secret details dummied out. If the template
contains database connection details, make sure *all of* account, password and
database id are dummied out so that installations with fake authentication
details do not lock up production databases by attempting to login with wrong
credentials.

Using the ``-a`` option does not require the original files to ever exist.
This can be used to trigger ``deploy_myapp_auth`` to run always for that
destination file. This is useful to generate the file contents from some
other already installed file.

If your application uses WMCore's security module with front-end
authentication, your service should just depend on installing ``wmcore-auth``.

mkproxy
^^^^^^^

The ``mkproxy`` does not take any arguments. It creates ``proxy`` sub-directory
under ``$project_state`` and records under ``$root/current/auth/proxy`` a note
for ``ProxyRenew`` to update a ``$project_state/proxy/proxy.cert``.

enable and disable
^^^^^^^^^^^^^^^^^^

The ``enable`` and ``disable`` do not take any arguments. They create and remove,
respectively, a ``$root/enabled/myapp`` flag to indicate the service *myapp* is
enabled or disabled on this host.

Command line options
^^^^^^^^^^^^^^^^^^^^

The ``Deploy`` script accepts the following command line options:

* ``-A arch`` installs for architecture _arch_ instead of the default.

* ``-M`` disables ``$MIGRATION`` commands.

* ``-R package@version`` is normally a required option. Normally this would
  be a meta-package version such as ``cmsweb@1207c``. For all ``deploy_pkg``
  commands which do not specify an explicit package version, the correct
  package version is automatically determined from the dependencies of the
  meta-package given here. A meta-package is a package which has no content,
  just dependencies, which is how we normally generate a consistent build and
  installation configuration for all the software for the entire cluster.

* ``-H host`` runs as if current host was *host*. This sets ``$host`` variable
  to the specified value instead of current host name.

* ``-a`` activates multi-account configuration and is used for production.
  Not using this option is no longer supported.

* ``-r comp=comp.pre`` overrides corresponding ``deploy_pkg`` repository
  parameter. Use this option to install pre-release or user-private builds. Do
  note *comp* and *comp.pre* have different package rebuild counts, so the
  ``-cmp*`` version suffixes vary. Hence ``-r`` override may not be enough,
  you may need to resort to editing ``deploy_pkg`` version arguments.

* ``-t version`` names the installation area *version* and points the
  ``current`` symlink to it. The production installations use the release
  name, such as ``hg1207c-frontend``.

* ``-p authdir`` instructs to use authentication secrets from directory
  *authdir*. This should contain subdirectories and files named by
  ``deploy_pkg -a`` options. Omitting this argument tells ``Deploy`` to
  generate dummy authentication info by invoking ``deploy_myapp_auth``
  actions.

* ``-s stages`` installs only the specified parts of recipes. It should be
  space separated list of words "prep sw post".

* ``-h`` shows help for the command.

The rest of the arguments are services to install, in the form
``service[@version][/variant]``. If ``@version`` is included, the version
overrides all package versions specified in deploy script and in the
meta-package dependencies. If the ``/variant`` is included, the
``$variant`` variable will have that value when executing the deploy
script functions.

Testing your deployment action
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In general you should be able to test your deployment action as follows.
Just edit the *Deploy* script locally and repeat as long as necessary. Use
``-r comp.pre`` or ``-r comp.$user`` to test with pre-release and private
builds. See :doc:`../environ/vm-setup` for details on how to set up an
environment for this. ::

    # Generally assumed working area, but can be anything.
    cd /data

    # Create secrets. Do this just once.
    mkdir -p auth/myapp
    vi auth/myapp/mysecret # whatever your app requires
    chgrp -R _sw auth
    chmod ug=r,o-rwx $(find auth -type f)
    chmod u=rwx,g=rx,o-rwx $(find auth -type d)

    # Grab configuration from github, in this case the HEAD of the master branch.
    # Edit 'deploy', 'manage', etc. files as you wish, then redeploy.
    git clone git://github.com/dmwm/deployment.git cfg

    # Basic standard deployment. Repeat as often as necessary. Note that
    # if you edit your 'manage' script or other configuration contents in
    # the github cloned area ($PWD/cfg), you normally need to re-run all
    # the three stages to install new files to the server area.
    $PWD/cfg/Deploy -a -p $PWD/auth -t mydev -s prep $PWD admin myapp
    sudo -H -u _sw bashs -lc "$PWD/cfg/Deploy -R cmsweb@1207c -a -p $PWD/auth -t mydev -s sw $PWD admin myapp"
    $PWD/cfg/Deploy -a -p $PWD/auth -t mydev -s post $PWD admin myapp

    # Same as above, but deploy from pre-release repository, using fake authentication.
    $PWD/cfg/Deploy -a -t mydev -s prep -r comp=comp.pre $PWD admin myapp
    sudo -H -u _sw bashs -lc "$PWD/cfg/Deploy -R cmsweb@1207c -a -t mydev -s sw -r comp=comp.pre $PWD admin myapp"
    $PWD/cfg/Deploy -a -t mydev -s post -r comp=comp.pre $PWD admin myapp

    # Override package version and variant when installing software.
    sudo -H -u _sw bashs -lc "$PWD/cfg/Deploy -R cmsweb@1207c -a -t mydev -s sw -r comp=comp.pre $PWD myapp@1.2.3/dev"

    # Start / stop / check server status.
    verb=status; for f in enabled/*; do
      app=${f#*/}; case $app in frontend) u=root ;; * ) u=_$app ;; esac; sudo -H -u $u bashs -lc \
      "$PWD/current/config/$app/manage $verb"
    done

    verb=start; for f in enabled/*; do
      app=${f#*/}; case $app in frontend) u=root ;; * ) u=_$app ;; esac; sudo -H -u $u bashs -lc \
      "$PWD/current/config/$app/manage $verb 'I did read documentation'"
    done

    verb=stop; for f in enabled/*; do
      app=${f#*/}; case $app in frontend) u=root ;; * ) u=_$app ;; esac; sudo -H -u $u bashs -lc \
      "$PWD/current/config/$app/manage $verb 'I did read documentation'"
    done
