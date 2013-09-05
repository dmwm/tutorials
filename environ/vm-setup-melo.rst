Local installation setup
---------------------

Local setup
^^^^^^^^^^^^^^^^^^^^^^^^

1. Request manager/frontend::
    # Script to get a working, single-user, mysql deploy of as many services as possible
    #   Things (like single-user) are technically cheating, but this script is intended to:
    #      1) Be testable. Single-user stuff is *much* easier to hook up to jenkins
    #      2) Be iterated. Once we get one variant working, the others are easier to follow
    #



    ###### CONFIG 
    # TODO: MAKE THIS CONFIGURABLE
    set -x
    set -e

    INSTALL_PWD=$PWD
    WMAGENT_INSTALL_POSTFIX=wmagent-install
    CMSWEB_INSTALL_POSTFIX=cmsweb-install
    WMAGENT_INSTALL_LOCATION=$PWD/$WMAGENT_INSTALL_POSTFIX
    CMSWEB_INSTALL_LOCATION=$PWD/$CMSWEB_INSTALL_POSTFIX
    WMCORE_SOURCE_TREE=/home/meloam/analysis/WMCore
    WMCORE_TAG=0.9.26
    CMSWEB_TAG=HG1302b
    MY_HOSTNAME=`hostname -f`

    REPO='-r comp=comp.pre.meloam'
    export SCRAM_ARCH=slc5_amd64_gcc461


    ###### CLEANUP REQMGR
    if [ -e $CMSWEB_INSTALL_LOCATION/current ]; then
      echo "Stopping existing ReqMgr services"
      $CMSWEB_INSTALL_LOCATION/current/config/reqmgr/manage stop 'I did read documentation' || true
      $CMSWEB_INSTALL_LOCATION/current/config/couchdb/manage stop 'I did read documentation' || true

      # stop frontend
      echo "Stopping frontend.."
      set +x
      (. $CMSWEB_INSTALL_LOCATION/current/apps/frontend/etc/profile.d/init.sh && httpd -f $CMSWEB_INSTALL_LOCATION/state/frontend/server.conf -k stop)
      set -x
    fi

    # temp
    rm -rf $CMSWEB_INSTALL_LOCATION

    # remove old cronjobs
    #crontab -r || true # dont die if no old crontab

    # stop old processes
    pkill -9 -f $CMSWEB_INSTALL_LOCATION || true

    ###### INSTALL REQMGR

    # skip sudo usage - just adds to /etc etc.
    # FIXME have this work both ways
    perl -p -i -e 's/.*sudo.*/:/' $PWD/cfg/frontend/deploy
    (
        mkdir -p $CMSWEB_INSTALL_LOCATION
        cd $CMSWEB_INSTALL_LOCATION
        # link to hostcert
        if [ ! -e certs ]; then
          mkdir -p certs
        fi
        if [ ! -r /etc/grid-security/hostcert.pem ]; then
            echo "ERROR: Grid certificates in /etc/grid-security must be readable!"
            echo "  (a future update will self-generate dummy certificates)"
        fi

        [ -e certs/hostcert.pem ] || $(sudo cp /etc/grid-security/hostcert.pem certs/hostcert.pem)
        [ -e certs/hostkey.pem ] || $(sudo cp /etc/grid-security/hostkey.pem certs/hostkey.pem)
        
        #[ -e certs/hostcert.pem ] || $(sudo cp ~/.globus/usercert.pem certs/hostcert.pem)
        #[ -e certs/hostkey.pem ] || $(sudo cp ~/.globus/userkey.pem certs/hostkey.pem)


        sudo chown `id -un`:`id -gn` certs/host*.pem
        chmod 600 certs/host*.pem
        
        export X509_USER_CERT=$PWD/certs/hostcert.pem
        export X509_USER_KEY=$PWD/certs/hostkey.pem
        
        # note single user install (which will be fixed)
        # need couchdb to get couchdb manage commands
        # need to support doing things with secrets
        $INSTALL_PWD/cfg/Deploy -R cmsweb@${CMSWEB_TAG} -t install -r comp=comp.pre -A $SCRAM_ARCH -s 'prep sw' $PWD admin frontend reqmgr workqueue asyncstageout
        
        # Hotpatch the RPM before we go any further
        if [ $WMCORE_SOURCE_TREE -a -e $WMCORE_SOURCE_TREE ]; then
            (
                set +x
                . current/apps/workqueue/etc/profile.d/init.sh && \
                cd $WMCORE_SOURCE_TREE && \
                wmc-dist-patch -s workqueue --skip-docs | grep -v -e '^copying' \
                                                                  -e '^creating' \
                                                                  -e '^byte-compiling'
            )
            (
                set +x
                . current/apps/reqmgr/etc/profile.d/init.sh && \
                cd $WMCORE_SOURCE_TREE && \
                wmc-dist-patch -s reqmgr --skip-docs | grep -v -e '^copying' \
                                                               -e '^creating' \
                                                               -e '^byte-compiling'
            )
        fi
        
        $INSTALL_PWD/cfg/Deploy -R cmsweb@${CMSWEB_TAG} -t install -r comp=comp.pre -A $SCRAM_ARCH -s post $PWD admin frontend reqmgr workqueue asyncstageout
        
       
        # get auth going

        # FIME this should be the host cert instead
        rm install/auth/reqmgr/*.pem
        rm install/auth/workqueue/*.pem
        ln -s $PWD/certs/hostkey.pem install/auth/reqmgr/dmwm-service-key.pem
        ln -s $PWD/certs/hostkey.pem install/auth/workqueue/dmwm-service-key.pem
        ln -s $PWD/certs/hostcert.pem install/auth/reqmgr/dmwm-service-cert.pem
        ln -s $PWD/certs/hostcert.pem install/auth/workqueue/dmwm-service-cert.pem
        #ln -s ~/.globus/userkey.pem install/auth/reqmgr/dmwm-service-key.pem
        #ln -s ~/.globus/userkey.pem install/auth/workqueue/dmwm-service-key.pem
        #ln -s ~/.globus/usercert.pem install/auth/reqmgr/dmwm-service-cert.pem
        #ln -s ~/.globus/usercert.pem install/auth/workqueue/dmwm-service-cert.pem


        # add in acdc couchapps - needed for integration tests
        #grep -q ACDC state/couchdb/stagingarea/acdc || echo "couchapp push $PWD/WMCore/src/couchapps/ACDC http://localhost:5984/wmagent_acdc" >> state/couchdb/stagingarea/acdc
        #grep -q GroupUser state/couchdb/stagingarea/acdc || echo "couchapp push $PWD/WMCore/src/couchapps/GroupUser http://localhost:5984/wmagent_acdc" >> state/couchdb/stagingarea/acdc
        
        # standard deploy assumes running as root - need to change a few things
        HTTPD_CONF=$CMSWEB_INSTALL_LOCATION/state/frontend/server.conf
        
        # run under current user
        perl -p -i -e 's/^User/#User/' ${HTTPD_CONF}
        perl -p -i -e 's/^Group/#Group/' ${HTTPD_CONF}
        
        # non-privileged port
        perl -p -i -e 's/Listen 80/Listen 8080/' ${HTTPD_CONF}
        perl -p -i -e 's/\*:80/\*:8080/' ${HTTPD_CONF}
        
        perl -p -i -e 's/Listen 443/Listen 8443/' ${HTTPD_CONF}
        perl -p -i -e 's/\*:443/\*:8443/' ${HTTPD_CONF}
     
        # non-rootowned key
        perl -p -i -e "s#SSLCertificateFile.*#SSLCertificateFile $CMSWEB_INSTALL_LOCATION/certs/hostcert.pem#" ${HTTPD_CONF}
        perl -p -i -e "s#SSLCertificateKeyFile.*#SSLCertificateKeyFile $CMSWEB_INSTALL_LOCATION/certs/hostkey.pem#" ${HTTPD_CONF}


        # add test cert credentials
        # TODO: FIX ME (what goes in extra-certificates.txt?)
        # drop existing stuff in the database


        cat ../extra-certificates.txt >> current/config/frontend/extra-certificates.txt
        cp ../sitedb-mapping.py . && chmod +x sitedb-mapping.py && (set +x ;. current/apps/reqmgr/etc/profile.d/init.sh && set -x && ./sitedb-mapping.py --mapping-file=../sitedb-mapping.txt current/auth/frontend/users.db )

        set -x

        echo "
    INSERT OR IGNORE INTO contact('surname', 'forename', 'username', 'dn') VALUES ('local','host','localhost','$dn');
    INSERT OR IGNORE INTO role('title') VALUES('admin');
    INSERT OR IGNORE INTO user_group('name') VALUES('reqmgr');
    INSERT OR IGNORE INTO group_responsibility('contact', 'role', 'user_group') VALUES ( (SELECT id FROM contact WHERE dn='$dn'), (SELECT id FROM role WHERE title='admin'), (SELECT id from user_group WHERE name='reqmgr') );
    INSERT OR IGNORE INTO group_responsibility('contact', 'role', 'user_group') VALUES ( (SELECT id FROM contact WHERE dn='$dn'), (SELECT id FROM role WHERE title='_admin'), (SELECT id from user_group WHERE name='CouchDB') );


    " | sqlite3 current/auth/frontend/users.db
        echo "INSERT INTO group_responsibility('contact', 'role', 'user_group') VALUES ( (SELECT id FROM contact WHERE dn='/DC=org/DC=doegrids/OU=People/CN=Andrew Malone Melo 788499'), (SELECT id FROM role WHERE title='admin'), (SELECT id from user_group WHERE name='reqmgr') );" | sqlite3 current/auth/frontend/users.db

        if [ -e ~/.globus/usercert.pem ]; then
            echo "
    INSERT OR IGNORE INTO contact('surname', 'forename', 'username', 'dn') VALUES ('local','user','localuser','$dn');
    INSERT OR IGNORE INTO group_responsibility('contact', 'role', 'user_group') VALUES ( (SELECT id FROM contact WHERE dn='$dn'), (SELECT id FROM role WHERE title='admin'), (SELECT id from user_group WHERE name='reqmgr') );
    INSERT OR IGNORE INTO group_responsibility('contact', 'role', 'user_group') VALUES ( (SELECT id FROM contact WHERE dn='$dn'), (SELECT id FROM role WHERE title='_admin'), (SELECT id from user_group WHERE name='CouchDB') );


    " | sqlite3 current/auth/frontend/users.db
        fi
        
        # update authmap
        (
        set +x
        source $CMSWEB_INSTALL_LOCATION/current/apps/frontend/etc/profile.d/init.sh && PYTHONPATH=$CMSWEB_INSTALL_LOCATION/current/auth/frontend:$PYTHONPATH $CMSWEB_INSTALL_LOCATION/current/config/frontend/mkauthmap -q -c sitedbread.db -o $CMSWEB_INSTALL_LOCATION/state/frontend/etc/authmap.json
        )
        # steal mysql from the wmagent installation (oracle isn't always around)
        # requiring oracle raises the bar of entry
        echo "
    import ReqMgrSecrets
    if getattr(ReqMgrSecrets, 'socket', None):
        config.reqmgr.database.socket = ReqMgrSecrets.socket
        config.CoreDatabase.socket = ReqMgrSecrets.socket
    " >> current/config/reqmgr/ReqMgrConfig.py
        WMAGENT_MYSQL_USER=`grep MYSQL_USER= $WMAGENT_INSTALL_LOCATION/WMAgent.secrets | sed 's#MYSQL_USER.*=##'`
        WMAGENT_MYSQL_PASS=`grep MYSQL_PASS= $WMAGENT_INSTALL_LOCATION/WMAgent.secrets | sed 's#MYSQL_PASS.*=##'`
        echo "connectUrl='mysql://$WMAGENT_MYSQL_USER:$WMAGENT_MYSQL_PASS@localhost/reqmgr'" >> current/auth/reqmgr/ReqMgrSecrets.py
        echo "socket='$WMAGENT_INSTALL_LOCATION/current/install/mysql/logs/mysql.sock'" >> current/auth/reqmgr/ReqMgrSecrets.py
        # we then need to install the databases
        (
            set +x
            source current/apps/reqmgr/etc/profile.d/init.sh 2>&1 /dev/null
            source $WMAGENT_INSTALL_LOCATION/current/apps/wmagent/etc/profile.d/init.sh 2>&1 /dev/null
            source $CMSWEB_INSTALL_LOCATION/current/apps/reqmgr/etc/profile.d/init.sh 2>&1 /dev/null
            export WMCORE_ROOT=current/apps/reqmgr/lib/python2.6/site-packages/
            export PYTHONPATH=$PWD/current/auth/reqmgr:$PYTHONPATH
            set -x
            mysql -e "DROP DATABASE IF EXISTS reqmgr" --password=$WMAGENT_MYSQL_PASS -u $WMAGENT_MYSQL_USER --socket=$WMAGENT_INSTALL_LOCATION/current/install/mysql/logs/mysql.sock

            mysql -e "CREATE DATABASE IF NOT EXISTS reqmgr" --password=$WMAGENT_MYSQL_PASS -u $WMAGENT_MYSQL_USER --socket=$WMAGENT_INSTALL_LOCATION/current/install/mysql/logs/mysql.sock
            current/apps/reqmgr/bin/wmcore-db-init  --config current/config/reqmgr/ReqMgrConfig.py --create --modules=WMCore.Agent.Database,WMCore.RequestManager.RequestDB
        )
        # couchdb on localhost:5984 until auth is fixed
        perl -p -i -e 's!"https://%s/couchdb.*!"http://localhost:5984"!' current/config/reqmgr/ReqMgrConfig.py
        perl -p -i -e 's!"https://%s/couchdb.*!"http://localhost:5984"!' current/config/workqueue/GlobalWorkQueueConfig.py
        perl -p -i -e 's!https://%s/reqmgr/reqMgr" % HOST!https://localhost:5984/reqmgr/reqMgr" % HOST!' current/config/workqueue/GlobalWorkQueueConfig.py
        # increase frequency of cron jobs - reduce testing turnaround
        (crontab -l && echo "*/2 * * * * $CMSWEB_INSTALL_LOCATION/current/config/workqueue/workqueue_task reqmgr_sync") | crontab -
        (crontab -l && echo "1-59/2 * * * * $CMSWEB_INSTALL_LOCATION/current/config/workqueue/workqueue_task housekeep") | crontab -

        # replace reqmgr
        perl -p -i -e 's!dev-cms-reqmgr.cern.ch!${MY_HOSTNAME}!' current/config/workqueue/GlobalWorkQueueConfig.py

        # get useful error messages
        perl -p -i -e 's!config.Webtools.environment.*!config.Webtools.environment="development"!' current/config/reqmgr/ReqMgrConfig.py
        # start couchdb
        current/config/couchdb/manage start 'I did read documentation'
        
        # start reqmgr - needs to be started with full path (else import error)
        (
            set +x
            # FIXME need mysql
            source $WMAGENT_INSTALL_LOCATION/current/apps/wmagent/etc/profile.d/init.sh
            set -x
            $CMSWEB_INSTALL_LOCATION/current/config/reqmgr/manage start 'I did read documentation'
        )

        # add values to reqmgr
        (
        set +x
        source $WMAGENT_INSTALL_LOCATION/current/apps/wmagent/etc/profile.d/init.sh
        source $CMSWEB_INSTALL_LOCATION/current/apps/reqmgr/etc/profile.d/init.sh
        export WMCORE_ROOT=$CMSWEB_INSTALL_LOCATION/current/apps/reqmgr/
        export PYTHONPATH=$PYTHONPATH:$CMSWEB_INSTALL_LOCATION/current/auth/reqmgr/
        export WMAGENT_CONFIG=$CMSWEB_INSTALL_LOCATION/current/config/reqmgr/ReqMgrConfig.py
        set -x
        cd $CMSWEB_INSTALL_LOCATION/current/apps/reqmgr/bin/
        ./requestDbAdmin add -v all
        ./requestDbAdmin list -v all
        )

        

        #start frontend (manage assumes root so cant be used) (again, to be fixed)
        set +x
        echo "Starting frontend.."
        ( . current/apps/frontend/etc/profile.d/init.sh && httpd -f $PWD/state/frontend/server.conf -k start )
        set -x
        # temporarily make a vomsmapfile for before cron fires
        dn=$(openssl x509 -noout -subject -in $CMSWEB_INSTALL_LOCATION/certs/hostcert.pem | cut -f2- -d\ )
        # not sure if this is even ncessary since we have extra-certificates.txt
        # but until it gets bootstrapped, things dont work
        echo "\"$dn\" cms" >> state/frontend/etc/voms-gridmap.txt
        if [ -e ~/.globus/usercert.pem ]; then
             dn=$(openssl x509 -noout -subject -in ~/.globus/usercert.pem | cut -f2- -d\ )
             echo "\"$dn\" cms" >> state/frontend/etc/voms-gridmap.txt
        fi


        curl -k --cert $CMSWEB_INSTALL_LOCATION/certs/hostcert.pem --key $CMSWEB_INSTALL_LOCATION/certs/hostkey.pem -X POST https://localhost:8443/reqmgr/admin/handleAddUser -d "user=meloam&email=andrew.melo@gmail.com"
        curl -k  --cert $CMSWEB_INSTALL_LOCATION/certs/hostcert.pem --key $CMSWEB_INSTALL_LOCATION/certs/hostkey.pem -X POST https://localhost:8443/reqmgr/admin/handleAddGroup -d "group=testing"
        curl -k   --cert $CMSWEB_INSTALL_LOCATION/certs/hostcert.pem --key $CMSWEB_INSTALL_LOCATION/certs/hostkey.pem -X POST https://localhost:8443/reqmgr/admin/handleAddToGroup -d "group=testing&user=meloam"
        curl -k   --cert $CMSWEB_INSTALL_LOCATION/certs/hostcert.pem --key $CMSWEB_INSTALL_LOCATION/certs/hostkey.pem -X POST https://localhost:8443/reqmgr/admin/handleAddTeam -d "team=TestingTeam"
        

    )

2. Install Workqueue
   try this::
    #
    # Script to get a working, single-user, mysql deploy of as many services as possible
    #   Things (like single-user) are technically cheating, but this script is intended to:
    #      1) Be testable. Single-user stuff is *much* easier to hook up to jenkins
    #      2) Be iterated. Once we get one variant working, the others are easier to follow
    #



    ###### CONFIG 
    # TODO: MAKE THIS CONFIGURABLE
    set -x
    set -e

    WMAGENT_INSTALL_POSTFIX=wmagent-install
    CMSWEB_INSTALL_POSTFIX=cmsweb-install
    WMAGENT_INSTALL_LOCATION=$PWD/$WMAGENT_INSTALL_POSTFIX
    CMSWEB_INSTALL_LOCATION=$PWD/$CMSWEB_INSTALL_POSTFIX
    WMCORE_SOURCE_TREE=/home/meloam/analysis/WMCore
    WMCORE_TAG=0.9.26
    REPO='-r comp=comp.pre.meloam'
    export SCRAM_ARCH=slc5_amd64_gcc461

    ###### CLEANUP

    # temp
    rm -rf $WMAGENT_INSTALL_LOCATION

    if [ -e $WMAGENT_INSTALL_LOCATION/current/config/wmagent/manage ]; then
      $WMAGENT_INSTALL_LOCATION/current/config/wmagent/manage stop-agent || true
      $WMAGENT_INSTALL_LOCATION/current/config/wmagent/manage stop-services || true
    fi

    rm -f $WMAGENT_INSTALL_LOCATION/current/install/mysql/logs/mysql.sock || true

    ###### AGENT (i.e. non-frontend compatible parts)
    # this becomes obsolete once WMAgent gets unified with the rest of the deployment

    mkdir -p $WMAGENT_INSTALL_LOCATION

    # stop old processes
    pkill -9 -f $WMAGENT_INSTALL_LOCATION || true

    # Dummy up some secrets
    # We have to use 5985 for the couch port because the cmsweb installation uses 5984
    # for couch

    # we have to bind couch to all IPs because not everyone listens when we tell them
    # where to connect

    WMAGENT_MYSQL_PASSWORD=`date | md5sum | awk '{ print $1 }'`
    WMAGENT_COUCH_PASSWORD=`date | md5sum | awk '{ print $1 }'`
    echo "
    MYSQL_USER=wmagent_user
    COUCH_USER=wmagent_user
    MYSQL_PASS=$WMAGENT_MYSQL_PASSWORD
    COUCH_PASS=$WMAGENT_COUCH_PASSWORD
    COUCH_PORT=5985
    COUCH_HOST=0.0.0.0
    WORKLOAD_SUMMARY_PORT=5985
    WORKLOAD_SUMMARY_HOSTNAME=`hostname -f`
    " >> $WMAGENT_INSTALL_LOCATION/WMAgent.secrets

    ( cd $WMAGENT_INSTALL_LOCATION
      export WMAGENT_SECRETS_LOCATION=$PWD/WMAgent.secrets
      /data/cfg/Deploy  -t wmagent-$WMCORE_TAG $REPO -R wmagent@$WMCORE_TAG -s "prep sw" $PWD wmagent@$WMCORE_TAG-comp

    if [ $WMCORE_SOURCE_TREE -a -e $WMCORE_SOURCE_TREE ]; then
        (
            set +x
            . $WMAGENT_INSTALL_LOCATION/current/apps/wmagent/etc/profile.d/init.sh && \
            which wmc-dist-patch &&
            cd $WMCORE_SOURCE_TREE && \
            wmc-dist-patch -s wmagent --skip-docs | grep -v -e '^copying' \
                                                            -e '^creating' \
                                                            -e '^byte-compiling'
        )
    fi
      /data/cfg/Deploy  -t wmagent-$WMCORE_TAG $REPO -R wmagent@$WMCORE_TAG -s post $PWD wmagent@$WMCORE_TAG-comp


      # force mysql to a reasonable size
      perl -p -i -e 's/set-variable = innodb_buffer_pool_size=2G/set-variable = innodb_buffer_pool_size=50M/' current/config/mysql/my.cnf
      perl -p -i -e 's/set-variable = innodb_log_file_size=512M/set-variable = innodb_log_file_size=20M/' current/config/mysql/my.cnf
      perl -p -i -e 's/key_buffer=4000M/key_buffer=100M/' current/config/mysql/my.cnf
      perl -p -i -e 's/max_heap_table_size=2048M/max_heap_table_size=100M/' current/config/mysql/my.cnf
      perl -p -i -e 's/tmp_table_size=2048M/tmp_table_size=100M/' current/config/mysql/my.cnf
      
      # get a configuration to modify
      current/config/wmagent/manage activate-agent
      
      # edit config - change default team and agent name
      perl -p -i -e 's/agentTeams =.*/agentTeams = "TestingTeam"/' current/config/wmagent/config-template.py
      perl -p -i -e 's/agentName =.*/agentName = "WMAgentTestInstance"/' current/config/wmagent/config-template.py
      
      # set a decent work dir
      # TODO should this be dereferenced?
      perl -p -i -e "s#workDirectory =.*#workDirectory = \"$WMAGENT_INSTALL_LOCATION/current/install/wmagent/workDir\"#" current/config/wmagent/config-template.py
      mkdir -p $WMAGENT_INSTALL_LOCATION/current/install/wmagent/workDir/ || true

      #reduce taskarchiver timeout
      perl -p -i -e 's/workflowArchiveTimeout =.*/workflowArchiveTimeout = 1/' current/config/wmagent/config-template.py

      # change retries
      perl -p -i -e 's/maxJobRetries =.*/maxJobRetries = 7/' current/config/wmagent/config-template.py


      # change to test workqueue
      # FIXME make the agent's cert right
      perl -p -i -e 's!https://cmsweb.cern.ch/couchdb/workqueue!https://se9.accre.vanderbilt.edu:8443/bigcouch/workqueue!' current/config/wmagent/config-template.py
      perl -p -i -e 's!centralWMStatsURL =.*!centralWMStatsURL = "https://se9.accre.vanderbilt.edu:8443/bigcouch/wmstats"!' current/config/wmagent/config-template.py

      
    # FIXME - need to bring the blank pickles along
      #Use the mock submitter plugin - https://svnweb.cern.ch/trac/CMSDMWM/wiki/HOWTOMockGrid
      #perl -p -i -e 's/CondorPlugin/MockPlugin/' current/config/wmagent/config-template.py
      perl -p -i -e 's/CondorPlugin/RemoteCondorPlugin/' current/config/wmagent/config-template.py
      echo "config.WorkQueueManager.pollInterval = 60" >> current/config/wmagent/config-template.py

      # reduce polling intervals
      perl -p -i -e 's/.pollInterval = .*/.pollInterval = 30/' current/config/wmagent/config-template.py

      # write to analysis DBS
      perl -p -i -e 's!config.DBSInterface.DBSUrl =.*!config.DBSInterface.DBSUrl = "https://cmsdbsprod.cern.ch:8443/cms_dbs_ph_analysis_02_writer/servlet/DBSServlet"!' current/config/wmagent/config-template.py

      # TODO FIXME - the blank pickles aren't on
      false && echo "
    config.BossAir.section_('MockPlugin')
    config.BossAir.MockPlugin.jobRunTime = 1
    config.BossAir.MockPlugin.mockPluginProcesses = 1
    config.BossAir.MockPlugin.fakeReport = '../../../../WMCore/test/python/WMCore_t/BossAir_t/FakeReport.pkl'
    config.BossAir.MockPlugin.lcFakeReport = '../../../../WMCore/test/python/WMCore_t/BossAir_t/LogCollectFakeReport.pkl'
    " >> current/config/wmagent/config-template.py

      # Add ASOTracker
      echo "
    config.component_('AsyncStageoutTracker')
    # The log level of the component. 
    config.AsyncStageoutTracker.logLevel = 'DEBUG'
    # The namespace of the component
    config.AsyncStageoutTracker.namespace = 'WMComponent.AsyncStageoutTracker.AsyncStageoutTracker'
    # maximum number of threads we want to deal
    # with messages per pool.
    config.AsyncStageoutTracker.maxThreads = 1
    # maximum number of retries we want for job
    config.AsyncStageoutTracker.maxRetries = 5
    # The poll interval at which to look for failed jobs
    config.AsyncStageoutTracker.pollInterval = 60
    config.AsyncStageoutTracker.couchurl     = 'http://127.0.0.1:5984/'
    config.AsyncStageoutTracker.couchDBName  = 'user_monitoring_asynctransfer' 

    " >> current/config/wmagent/config-template.py
      # stop old processes
      pkill -9 -f $PWD || true
      $PWD/current/config/wmagent/manage stop-services || rm -f current/install/mysql/logs/mysql.sock
      $PWD/current/config/wmagent/manage stop-agent || true
      pkill -9 -f $PWD || true
      
      current/config/wmagent/manage start-services
      rm -f current/install/wmagent/.init
      current/config/wmagent/manage init-agent
      #current/config/wmagent/manage execute-agent wmagent-resource-control --add-all-sites --plugin=MockPlugin --running-slots=100
      # this has to be after the init-agent because the deployment stuff doesn't
      # line up
      perl -p -i -e 's!centralWMStatsURL =.*!centralWMStatsURL = "https://se9.accre.vanderbilt.edu:8443/bigcouch/wmstats"!' current/config/wmagent/config.py
      current/config/wmagent/manage start-agent
      current/config/wmagent/manage execute-agent wmagent-resource-control --site-name=T2_US_Vanderbilt --cms-name=T2_US_Vanderbilt --pending-slots=1000 --running-slots=1000 --ce-name=ce1.accre.vanderbilt.edu --se-name=se1.accre.vanderbilt.edu --plugin=RemoteCondorPlugin
      for JOBTYPE in Analysis Merge Processing Production LogCollect Cleanup; do
        current/config/wmagent/manage execute-agent wmagent-resource-control --site-name=T2_US_Vanderbilt --pending-slots=1000 --running-slots=1000 --task-type=$JOBTYPE
      done

    )
3. update and reboot
   hit this to update with your source tree::
    #!/bin/bash

    INSTALL_PWD=$PWD
    WMAGENT_INSTALL_POSTFIX=priority-install
    CMSWEB_INSTALL_POSTFIX=cmsweb-install
    WMAGENT_INSTALL_LOCATION=$PWD/$WMAGENT_INSTALL_POSTFIX
    CMSWEB_INSTALL_LOCATION=$PWD/$CMSWEB_INSTALL_POSTFIX
    WMCORE_SOURCE_TREE=/home/meloam/analysis/WMCore

    if [ -e $WMAGENT_INSTALL_LOCATION/current ]; then
        echo "Stopping existing wmagent processes"
        $WMAGENT_INSTALL_LOCATION/current/config/wmagent/manage stop-agent || true
    fi

    if [ $WMCORE_SOURCE_TREE -a -e $WMCORE_SOURCE_TREE ]; then
        (
            set +x
            . $WMAGENT_INSTALL_LOCATION/current/apps/wmagent/etc/profile.d/init.sh && \
            which wmc-dist-patch &&
            cd $WMCORE_SOURCE_TREE && \
            wmc-dist-patch -s wmagent --skip-docs | grep -v -e '^copying' \
                                                            -e '^creating' \
                                                            -e '^byte-compiling'
        set -x
        export WMAGENT_SECRETS_LOCATION=$WMAGENT_INSTALL_LOCATION/WMAgent.secrets
        export WMAGENT_CONFIG=$WMAGENT_INSTALL_LOCATION/current/config/wmagent/config.py
        if [ -d $WMAGENT_INSTALL_LOCATION/wmagent-0.9.26/sw.pre.meloam/slc5_amd64_gcc461/external/yui/2.9.0 ]; then
            export YUI_ROOT=$WMAGENT_INSTALL_LOCATION/wmagent-0.9.26/sw.pre.meloam/slc5_amd64_gcc461/external/yui/2.9.0
            which wmagent-couchapp-init
            wmagent-couchapp-init;
        else
            echo "Need to reset YUI_ROOT"
            exit 1
        fi
        unset WMAGENT_CONFIG

        $WMAGENT_INSTALL_LOCATION/current/config/wmagent/manage start-agent
        )
    fi
