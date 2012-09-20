Writing a manage script
-----------------------

Overview
^^^^^^^^

This page describes how to maintain DM/WM software *manage* script.
All the code resides in DM/WM github `deployment <https://github.com/dmwm/deployment>`_.
For each service there is a sub-directory which contains the deployment script
`deploy <ops-deploy.html>`_, server management script `manage <ops-manage.html>`_, and
any configuration files except security-sensitive authnetication data.

The scripts are designed to resemble linux /etc/init.d service management scripts.

(Not) making changes to the scripts
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Please do not commit any changes directly. Please submit tested changes as per
`dev-git <../environ/dev-git.html>`_ guidelines. All changes
pertaining to the service -- `deploy scripts <ops-deploy.html>`_, manage scripts,
`monitoring specs <ops-monitor.html>`_ and server configuration files -- should
be supplied in the patch series. We will review your changes, and may request
changes. Once the changes are approved, we will commit them to github for you.
You should assume progressive clean-up of your deployment action can happen
on every new server version submission.

How the management scripts is used
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The management scripts are used to *start* and *stop* services, and to get
service *status*. Where appropriate the script should allow specific server
sub-components to be controlled separately. The script also provides a
succinct *help* to give users an overview of what it can do; more complete
details will should be given on service-specific wiki pages. The management
operations should be specific to one exact service, not all services of that
kind; see below for implications on this.

How to run the management script
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Simply run the ``manage`` script, providing the action, and for start/stop
actions a security string. The security string should be something which
ensures the operator is very likely to be following copy-and-paste
instructions from service documentation, not executing random scripts on
the system which look executable. Examples on using the management script:

* Get help or status, or some other action not affecting the service - no
  security string checking: ::

      sudo -H -u _app bashs -lc "/data/current/config/app/manage help"
      sudo -H -u _app bashs -lc "/data/current/config/app/manage version"
      sudo -H -u _app bashs -lc "/data/current/config/app/manage status"

* Start, stop or any other action which affects the service - must use a
  security string check: ::

      sudo -H -u _app bashs -lc "/data/current/config/app/manage restart 'security string'"
      sudo -H -u _app bashs -lc "/data/current/config/app/manage start 'security string'"
      sudo -H -u _app bashs -lc "/data/current/config/app/manage stop 'security string'"

What the script should and should not do
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The management script should do everything required to start, stop and get the
status of the service it operates, but nothing else. The script should not touch
the installation or configuration files in any way. It can however modify server
state files, such as reload server configuration after an upgrade. In production
the script will run under application-specific daemon account which can only read
but not write to the software area and the configuration files.

Typically the script prepares the environment for the service, and then runs
commands affecting the service: to start it, to stop it, or to check its
status. More complete requirements are listed below:

* The script should operate only on one exact service corresponding to the
  ``config`` directory location. Other processes running under the user's
  account not related to the service in question should never be affected
  in any way. Specifically it should be possible to run several instances of
  a server under one account. In other words, the server and management script
  must agree on a reliable mechanism to select process to act on, such as
  saving a pid file.

* Try your best to make everything in the script location, user and server
  independent. Ideally it would work for any user in any installation path on
  any server system. This will make testing and development considerably
  easier. We understand sometimes it's necessary to hard-code specific
  details, but in our experience services hard-code too much and location
  independence is quite easy to achieve.

* Management script should be a bourne shell script. Usage of BASH, KSH or
  ZSH extensions is discouraged. CSH derived scripts are not accepted.

* The script should resemble linux /etc/init.d service management scripts.

* The minimum set of actions every service must support are: start
  (= restart), stop, status, version and help

* The *start* action should always be written as *restart*, i.e. to stop any
  existing running instance first.

* The script may provide additional actions provided they are described in
  help and on the service wiki page.

* Actions which start or stop service or its parts, or otherwise affect
  running services, must require a security string command line option to
  ensure minimal familiarity with service documentation. Many services use
  "I did read documentation" as the string and reminder to operators.

* Keep server environment to minimum and as clean as possible. Use ``export``
  only to set values which are communicated to the server process.
  Everything else should be normal, non-exported, shell variables.

* Keep the help succinct and clear, give additional information on service
  wiki page. Use existing management scripts as an example; the help should
  be written into the script itself as comments starting with ``##H``.

* Do not create additional external scripts from the management script. All
  scripts must be created at deployment step, either from RPM (site and
  location invariant parts) or installed from github or CVS (CERN instance
  specific parts).

* Avoid calling intermediate server launch scripts, instead embed the actions
  directly to your management script for transparency.

* If you need host-specific configuration, use the host name to select the
  right values to use. For example pick from files by logical purpose (dev,
  pre-prod, prod) by host name; don't use the host name itself for file
  names. We rather prefer the configuration files are host independent if at
  all possible. Any server using python configuration files can certainly do
  this internally.

* Server logs should be written to ``/data/logs/<name>``. The directory
  should be created during deployment. See `ops-deploy  <ops-deploy.html>`_.

* Server logs should be automatically rotated daily. We prefer you pipe server
  output via ``rotatelogs`` to achieve this. Specifically we prefer all server
  output to be sent to standard output then piped to ``rotatelogs``, rather than
  using any internal log rotating schemes.

* Avoid server version selection logic. Use the symlink created by
  `deploy_pkg` at installation time to decide
  which version of the server to act on. If we need to swap running server
  versions, we can just redirect the link.

* The script should always be tested to work for the newest service version,
  plus any versions still in pre-production or production use.

Use the template below for a new management script.

Testing the management script
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Always test new scripts in your private test installation, preferably using a
`developer virtual machine <../environ/vm-setup.html>`_ with realistic
multi-account setup.
Exercise all actions supported by your management script and make sure they
perform the desired actions and nothing else. You can use
``sh -x manage <options>`` to be sure of exact details executed. Verify it
doesn't attempt to operate servers not under its management.

It is highly recommended to verify the following deployment / reboot / server
management combinations all work. The server should restart automatically on
each reboot, and should do the "right thing", e.g. if rebooted after an
upgrade.

1. stop/reboot (no upgrade)
2. stop/upgrade/reboot (no start/stop)
3. stop/upgrade/start/stop/reboot
4. stop/upgrade/start/reboot (no stop)
5. upgrade/reboot (no stop, no start/stop)
6. reboot (no stop, no upgrade)

Template management script
^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    #!/bin/sh

    ##H Usage: manage ACTION [SECURITY-STRING]
    ##H
    ##H Available actions:
    ##H   help        show this help
    ##H   version     get current version of the service
    ##H   status      show current service's status
    ##H   sysboot     start server from crond if not running
    ##H   restart     (re)start the service
    ##H   start       (re)start the service
    ##H   stop        stop the service
    ##H
    ##H For more details please refer to operations page:
    ##H   https://twiki.cern.ch/twiki/bin/view/CMS/<twiki-page>

    if [ $(id -un) = cmsweb ]; then
      echo "ERROR: please use another account" 1>&2
      exit 1
    fi

    ME=$(basename $(dirname $0))
    TOP=$(cd $(dirname $0)/../../.. && pwd)
    ROOT=$(cd $(dirname $0)/../.. && pwd)
    CFGDIR=$(dirname $0)
    LOGDIR=$TOP/logs/$ME
    STATEDIR=$TOP/state/$ME
    COLOR_OK="\\033[0;32m"
    COLOR_WARN="\\033[0;31m"
    COLOR_NORMAL="\\033[0;39m"

    . $ROOT/apps/<app-name>/etc/profile.d/init.sh

    # Start service conditionally on crond restart.
    sysboot()
    {
      if [ $(pgrep -u $(id -u) -f "<PROCESS-PATTERN>" | wc -l) = 0 ]; then
        start
      fi
    }

    # Start the service.
    start()
    {
      cd $STATEDIR
      echo "starting $ME"
      <RUN-THE-SERVER> </dev/null 2>&1 |
        rotatelogs $LOGDIR/<APP>-%Y%m%d.log 86400 >/dev/null 2>&1 &
    }

    # Stop the service.
    stop()
    {
      echo "stopping $ME"
      for PID in $(pgrep -u $(id -u) -f "<PROCESS-PATTERN>" | sort -rn); do
        PSLINE=$(ps -o pid=,bsdstart=,args= $PID |
                 perl -n -e 'print join(" ", (split)[0..6])')
        echo "Stopping $PID ($PSLINE):"
        kill -9 $PID
      done
    }

    # Check if the server is running.
    status()
    {
      pid=$(pgrep -u $(id -u) -f "<PROCESS-PATTERN>" | sort -n)
      if [ X"$pid" = X ]; then
        echo -e "$ME is ${COLOR_WARN}NOT RUNNING${COLOR_NORMAL}."
      else
        echo -e "$ME is ${COLOR_OK}RUNNING${COLOR_NORMAL}, PID" $pid
      fi
    }

    # Verify the security string.
    check()
    {
      CHECK=$(echo "$1" | md5sum | awk '{print $1}')
      if [ $CHECK != 94e261a5a70785552d34a65068819993 ]; then
        echo "$0: cannot complete operation, please check documentation." 1>&2
        exit 2;
      fi
    }

    # Main routine, perform action requested on command line.
    case ${1:-status} in
      sysboot )
        if ps -oargs= $PPID | grep -q crond; then
          sysboot
        else
          echo "$0: sysboot is for cron only" 1>&2
          exit 1
        fi
        ;;

      start | restart )
        check "$2"
        stop
        start
        status
        ;;

      status )
        status
        ;;

      stop )
        check "$2"
        stop
        status
        ;;

      help )
        perl -ne '/^##H/ && do { s/^##H ?//; print }' < $0
        ;;

      version )
        echo "$<PROJECT-NAME>_VERSION"
        ;;

      * )
        echo "$0: unknown action '$1', please try '$0 help' or documentation." 1>&2
        exit 1
        ;;
    esac
