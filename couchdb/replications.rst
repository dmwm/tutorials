Setting up CouchDB replications
_______________________________

This section shows up the supported way of setting up couchdb
replications. It has been validated by the http group, and other possible
setups are discouraged.


Replications vs Accessing docs directly
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
While designing an application that needs the data stored in the central
couch, one may wonder why using a replication and not reading or writing
docs directly to the central couch.

Using replications has the advantages:

* You can remove the complexity of error handling and recovering
  from failures from your agent code since the couchdb takes care of all of
  that automatically for you;
* Pooling data locally is easier than remotely.

On the other hand, you would need to be able to deploy/manage a local
couchdb instance. Therefore, if it is just a few summary documents
you need to collect centrally and you can
safely miss some in case of failures, then just make each agent do a
normal HTTP GET/PUT request directly into the central DB and get rid
of the local couch machinery at all. However, if you already have a local
couch for whatever purpose, you should profit from that.


Key design decisions
^^^^^^^^^^^^^^^^^^^^

1. Do always use bidirectional replication

   Even on cases where your application is designed to only get updates in
   one direction, you should always use bidirection replication.

   CouchDB internally keeps update sequencies in both endpoints.
   If you loose the state in the source the endpoints for whatever reason
   like when you wipe out one of the deployed area), couchdb will restart the
   update_seq and will fool the other couch endpoint, thus not picking up any
   updates until you reach the same update_seq you had before. Consider
   for instance the situation below for a unidirectional replication A->B:

   * endpoint A is in upd_seq #123 -> endpoint B is in upd_seq #123
   * A gets wiped out or loose state for whatever reason
   * endpoint A is in #0 -> endpoint B is in upd_seq #123
   * B will not get any updates until A reaches #123 again

   This is potentially undesirable when you replicate from offsite
   agents to the central one. Since you can have several agents, chances
   of loosing the state in one of them is higher.

2. Replications are always defined offsite

   When defining the replications in the deploy script, do always set them
   only for the offsite couch variant. This is what is known as the "push
   mode". The central couch hosted in CMSWEB
   cannot authenticate against the offsite instance and additionally the
   authorization rules are always defined centrally to be under control.
   The replication engine in the version 1.1.0 currently in use issues
   all write operations internally as _admin, so the central instance cannot
   forbit to write documents coming from outside. On the other hand,
   replications setup offsite can only write centrally if they are authorized
   by the validate_doc_update function in the central database. The offsite
   couchdb will use a proxy or service certificate when accessing the central
   one, thus getting authenticated through the CMSWEB front-ends.


3. Do not replicate design documents

   Design documents should always be filtered out from the replications
   because they are pretty much an application. Allowing it to replicate
   means that offsite deployments could "upgrade" the couchapp without
   going through the standard deployment procedures, therefore not tracking
   the changes in any VCS.

   This is specially critical in cases where the design document is
   changed unintentionally, such like due to bugs in the offsite agent,
   or mistakes by its operator running some cleanup procedure.


4. Use the _replicator DB, not the _replicate feature

   The _replicate and the _replicator are two independent replication engines
   inside couchdb. Both had its own set of bugs that were affecting us.
   For instance, see this query on _replicate bugs in the CouchDB
   `issue tracker <https://issues.apache.org/jira/secure/IssueNavigator.jspa?reset=true&jqlQuery=project+%3D+COUCHDB+AND+%28summary+~+_replicate+OR+description+~+_replicate%29+AND+issuetype+%3D+Bug>`_.

   At some point, the collaboration had to decide to provide support
   (fixing the bugs) of
   either one or another, because it would be a waste of manpower
   to keep stable releases of both. It was than opted by the _replicator
   solution because it can keep replications persistent. Therefore,
   the couchdb RPMs only include patches to the _replicator related bugs,
   and you should stick with it.


5. Not all replication features are supported

   As per the same reason why _replicate is not supported, only a subset
   of the _replicator DB features is supported. To the known use cases
   of replication inside DMWM, however, the subset of features below is
   enough.

   * source
   * target
   * continuous
   * filter

   Other features like non-continous replications, proxy, query_params,
   doc_ids, etc have not been validated. In fact, you'll always stick
   with the supported features if following the setup intructions below
   in this document.


The setup
^^^^^^^^^

While defining the dependencies in your deploy script, make it require
the ``couchdb offsite`` variant: ::

   deploy_myapp_variants="default offsite"
   deploy_myapp_deps() {
      deploy couchdb $variant
   }

Then put this in post section: ::

   (echo {,https://centralhostname/couchdb/}mydbname mydesigndocname/repfilter
    echo {https://centralhostname/couchdb/,}mydbname mydesigndocname/repfilter
   ) > $root/state/couchdb/replication/myapp

Where the central host name will mostly likely be ``cmsweb.cern.ch``. Each
``echo`` represents a *source_db_url* *target_db_url* *replication_filter_fucntion* tuple.
At the bare minimum, the replication filter should contain: ::

   "repfilter": "function(doc, req) {
     return ! doc._id.match('_design/(.*)')
   }"

This is already the case when using the couchskel app as a template for
starting your new couchapp. There are cases, however, where you might want to
extend such a filter so that you replicate only the documents relevant to your
offsite application.
