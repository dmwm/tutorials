Writing monitoring specs
------------------------

Overview
^^^^^^^^

This page describes how to maintain DM/WM software monitoring
specifications. All the code resides in DM/WM github
`deployment <https://github.com/dmwm/deployment>`_. For each service there
is a sub-directory which contains the deployment script
`deploy <ops-deploy.html>`_, server management script `manage <ops-manage.html>`_,
monitoring specs and any configuration files except security-sensitive
authnetication data.

(Not) making changes to the scripts
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Please do not commit any changes directly. Please submit tested
changes as pull requests as per `dev-git <../environ/dev-git.html>`_ guidelines.
All changes pertaining
to the service -- `deploy scripts <ops-deploy.html>`_,
`manage scripts <ops-manage.html>`_, monitoring specs and server configuration
files -- should be supplied in the pull request. We will review your patch,
and may request changes. Once they are approved, we will commit them to
github for you. You should assume progressive clean-up of your deployment
action can happen on every new server version submission.

How the monitoring specs are used
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A central ``ServerMonitor`` script scans project configuration directories
``$root/current/config/*`` for files matching name ``monitoring*.ini``. It
reloads any recently changed configuration files, and probes all servers
every 30 seconds or so. The monitor looks for three things: whether server
processes are running, whether server logs have any error messages, and
whether the server responds to HTTP requests. Each server monitor file
specifies how to monitor one server.

The server monitor collects all this data and feeds it into a log file which
is then scanned by site monitoring tools, at CERN the lemon system. If
expected output isn't found in this feed, alarms will eventually be raised,
operators notified and action taken. In order to have a server properly
monitored the following are needed:

* the monitoring spec files
* lemon alarms and exceptions defined in `quattor <https://sls.cern.ch/cdb-tpl-view/tpl_view.php?profile=prod/customization/cms/webtools/backend/lemon&os=slc5&arch=x86_64&svcclass=vocms&resource=cms&customization=webtools/backend>`_
* `operations procedures <https://cern.ch/cms-http-group/ops-alarms.html>`_ for operators
* `service critical description <https://twiki.cern.ch/twiki/bin/viewauth/CMS/CMSCriticalServicesDocumentation>`_

Specifying monitoring
^^^^^^^^^^^^^^^^^^^^^

The service can have one or more monitoring spec files. If it has just one,
it should be called ``monitoring.ini``, and in this case the "stem" is the
project directory name, *myapp* for ``$root/current/config/myapp/monitoring.ini``.
If it has several, the files should be called ``monitoring-*.ini``, and the
"stem" is whatever is matched by the star. The stem is what is used to
report monitoring results in the output feed.

The file can have the following definitions:

* ``LOG_FILES`` defines a glob pattern for server log files, relative to
  ``$root/logs/myapp`` log directory. Normally this is just ``*.log``.

* ``LOG_ERROR_REGEX`` defines a perl regular expression to search in the
  log files. Any lines which match the pattern are reported as errors.

* ``PS_REGEX`` defines a perl regular expression to search against process
  command lines. PIDs for processes matching this expression are reported as
  live for this server. Be careful to be sufficiently restrictive in order
  not to match unexpected processes, such as running editors on server
  configuration files!

* ``PING_URL`` defines an URL to ping on the server, typically
  ``http://localhost:12345/path``. The URL shouldn't be too expensive on the
  server as it will be accessed quite frequently, but it should exercise
  enough of the server to make sure it's properly live. Avoid URLs which result
  in redirections; although redirections are followed, there is danger of
  many redirects, especially infinite recursion. The server monitor protects
  itself against infinite URL recursion, but it will still submit a rapid fire
  sequence of some requests every 30 seconds if the URL is bad.

* ``PING_REGEX`` defines a perl regular expression matched against the entire
  HTTP response. The server is reported to be alive only if the response
  matches this regular expression. The response includes all HTTP response
  headers as well as the body. Be careful with the regexp if the response is
  binary data.

Example
^^^^^^^

::

    LOG_FILES='*.log'
    LOG_ERROR_REGEX='^Traceback .most recent|\] HTTP Traceback'

    PS_REGEX="Root.py.*[/]T0MonConfig.py"

    PING_URL="http://localhost:8300/"
    PING_REGEX=".*T0Mon main page.*"

Typical monitoring feed
^^^^^^^^^^^^^^^^^^^^^^^

You can check for lemon errors with ``sudo /usr/sbin/lemon-host-check -sh``.
If there are no exceptions, the output will look like this: ::

    $ sudo /usr/sbin/lemon-host-check -sh
    [INFO] lemon-host-check version 1.3.3 started by root at Mon Feb  28 12:26:23 2011 on vocms106.cern.ch
    [VERB] Exceptions: 0 - Running actuators: 0 - Disabled exceptions: 0 - State: Production

The typical monitoring feed in ``$root/logs/admin/feed.lemon`` looks like a
series of these. ::

    Feb 28 12:10:11 t0mon PING OK <http://localhost:8300/>
    Feb 28 12:10:12 t0mon PROCESS pid:27446

If the server isn't running, then ``PROCESS`` lines are simply missing and
lemon will eventually raise an alarm. A failed ``PING`` might look like this: ::

    Feb 28 12:10:07 phedex-web PING FAILED url:<http://localhost:7101/phedex>

An error scanned from server logs might look like this: ::

    Feb 28 12:10:07 sitedb ERROR file:<.../sitedb-20110228.log>:50149 line:<[snip] error: Traceback (most recent call last):>
