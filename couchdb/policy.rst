Policies for CouchDB based applications
----------------------------------------------

1. All data is read public to CMS users

   All the information stored in the CMSWEB couchdb is readible from
   any CMS authenticated user (anyone with an hypernews account that has been
   included in the CMS VO). Applications that require better control of whom
   should or not read the data should not use the CMS couchdb as a storage
   back-end.


2. Create/drop couch databases operations are not allowed
   All the couch databases must be defined during deployment time, so that
   the couch manage scripts can push them at startup time.

   Although the service responsibles can still
   login to the back-end machines and fire create/drop requests directly
   through the localhost interface, they are not allowed to do so.
   The localhost access is intended to be used only by the cluster admins
   for special operations and by the couch manage scrips.

   The database drop is forbidden too, and if a database needs to be
   deleted, fire a special request to the HTTP group.


3. Document creation, modification and deletion

   The couchapps hosted in CMSWEB should control which CMS users
   have write permissions to do what. That means it should always
   come with a validate_doc_update function as per the skeleton
   couchapp defining which roles/sites/groups are allowed
   to create, modify and delete documents. Global couchdb admins
   should always be allowed to do such operations.


4. Offsite deployments of couchdb are not allowed to expose the replicated data

   All the non CMSWEB couchdb instances should only be accessible trough the
   localhost interface. Ops and other systems willing to query them, should be
   configured to do the proper tunneling. The offsite couch instances should
   be by no means be made public, nor use passwords or hold local CMS user auth
   information grabbed from elsewhere.
