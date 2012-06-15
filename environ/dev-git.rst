Working with git for CMS DMWM projects
--------------------------------------

.. _dev-git-part-1:

Creating feature branches and making a pull request
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

1. Create an account at github. From here on the instructions assume
   your github account is called 'mygit'. Replace that string below
   in all the instructions with your real github account name. Login
   to your github account on the website. Set up your github SSH keys.

2. Locate the code repository you work on. This could for example be
   'deployment' repository from 'dmwm' user, or 'WMCore' repository from
   'geneguvo'. On github web site, fork the repository to your
   own account. These instructions assume the 'deployment' repository
   from 'dmwm' account.

3. Clone the repository to your local area, and set up 'upstream' git
   remote for the original, and a tracking branch 'upstream' for the
   upstream's master branch. ::

       cd My/Dev/Area
       git clone git@github.com:mygit/deployment.git
       cd deployment

       git remote add upstream git://github.com/dmwm/deployment.git
       git fetch upstream
       git branch --track upmaster remotes/upstream/master

4. To start a new development, first update the 'upmaster' branch to
   current tip of the upstream repository. ::

       git checkout upmaster
       git pull

5. Create a new feature branch for your development. We'll call this
   'hg1206-reqmgr-update', obviously you should pick a name appropriate
   for your changes. ::

       git checkout -b hg1206-reqmgr-update

6. Modify the code and commit. If you like using 'stg', you can and
   should use it here to manage your patch stack. In that case, at the
   beginning say 'stg init' to initialise your branch for stg, then at
   the end when you are ready to submit, do 'stg commit -a' to commit
   the patches on your branch. The following assumes raw git use. ::

       vi reqmgr/deploy
       git add reqmgr/deploy
       git commit -m "Incredibly awesome changes."

7. Merge with upstream, in case there were changes in the mean time.
   Resolve any conflicts which arise. Note that you go to the tracking
   branch to do the pull, and merge locally from that branch to your
   development branch. You should also keep your 'master' branch up to
   date so on github web site it's easier to see the differences in
   your branch. In this example, you never commit anything to 'master',
   you just merge changes from 'upmaster'. ::

       git checkout upmaster
       git pull

       git checkout master
       git merge upmaster

       git checkout hg1206-reqmgr-update
       git merge upmaster

8. Push the changes to your github account. The 'origin' is where you did
   the initial clone, i.e. the 'deployment' repository in *your* github
   account. You should push both your 'master' (see above why) and all the
   feature branches you worked on. This is your last change to review the
   changes before publishing them. ::

       # review history
       git log --date-order --pretty=oneline --decorate --graph

       # review with full details, including patches
       git log -p

       git push origin master hg1206-reqmgr-update

9. At github web site, select your account's 'deployment' repository, and
   the 'hg1206-reqmgr-update' branch, and follow the pull request recipe
   to create a pull request for 'dmwm'. ::

   <http://help.github.com/send-pull-requests/>


.. _dev-git-part-2:

Applying a pull request after a review
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

1. First review the pull request on github's web site. It has all the tools
   we previously used on SVN Trac for commenting and iterating on patches.
   Once you're happy with the patch, proceed with the following.

2. Clone your own repository from github. ::

       cd My/Dev/Area
       git clone git@github.com:mygit/deployment.git
       cd deployment

3. Go on your master branch, and pull if necessary. ::

       git checkout master
       git pull

4. Add a remote for the person who submitted the pull request. We assume the
   user is called 'joebloggs', who cloned the repository under the same name.
   We create the remote named by the user. ::

       git remote add joebloggs git://github.com/joebloggs/deployment.git
       git fetch joebloggs

5. Merge the change set that was submitted and push to your own repo. If
   necessary, you may want to squash the patches together here, if the
   original author has not created single atomic commits per feature as
   they should have. See 'git merge --squash' documentation for details. ::

       git merge joebloggs/hg1206-reqmgr-update
       git push origin master

6. Note that other users go back into :ref:`dev-git-part-1`, step 4 to pull
   the changes you have just committed. Also note that git has no single
   master, so in fact anyone can execute the steps in :ref:`dev-git-part-2`
   and push to their own repo, and anyone else can merge those changes. This
   means work can proceed from any repository. In all likelihood we will try
   to keep repositories under 'dmwm' user with current state of the art at all
   times, with multiple committers with the necessary rights.


.. _dev-git-part-3:

Converting CMSDMWM SVN repository to github git repository
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

1. Clone the SVN repository using git, as per PatchManagement instructions.
   If you already have such a working area, you can use it, but make sure
   there are no uncommitted changes there. ::

       cd My/Dev/Area
       git svn clone svn+ssh://svn.cern.ch/reps/CMSDMWM/SiteDB -s


2. GIT-SVN tags are not real git tags but branches, so to preserve them you
   need to extract the version they were attached to. In conversions we have
   done, the tag parent commit is always the version that was tagged, so it
   can be designated with "revision^" in git parlance. If you only want some
   of the tags preserved, add a "grep" filter in command below. ::

       cd SiteDB
       git branch -a -l -v | grep remotes/tags |
         awk '{print substr($1, 14), $2}' |
         while read tag cid; do echo git tag $tag $cid^; done

   If the output of the above command looks reasonable to you, rerun the
   command piping the output to sh: "git branch ... ; done | sh -x".

3. Create a parallel directory for your pure-git conversion. We'll call
   the github area with "GH" prefix to distinguish it. We'll call the
   remote as 'svn' to avoid confusingly naming it 'master'. ::

       cd .. # Back to My/Dev/Area
       git clone -o svn SiteDB GHSiteDB
       cd GHSiteDB

4. Review that the tags are now correctly listed for all history. The
   --decorate option to 'git log' should be adding them to the listing. ::

       git log --date-order --pretty=oneline --decorate --graph

5. Create an empty repository on github, e.g. here 'sitedb'. Then add that
   repository as a reference to your converted repository, and push it all
   there. Note that we're still on the 'svn' branch we created initially.
   If you are going to reuse this repository after conversion, you may want
   to call it 'origin' instead of 'github' for future convenience. ::

       git remote add github git@github.com:mygit/sitedb.git
       git push --tags -u github svn


.. _dev-git-part-4:

Converting filtered CMSDMWM SVN repository
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If you want to execute the instructions above, but want to split your repository
so it becomes multiple git repositories, you'll want to use 'git filter-branch'
to extract only the parts you want. For example the following is how we extracted
'Infrastructure/Deployment' to its own git repository: ::

     # clone svn repository and make separate work area
     cd My/Dev/Area
     git svn clone svn+ssh://svn.cern.ch/reps/CMSDMWM/Infrastructure -s
     git clone -o svn Infrastructure GHDeployment
     cd GHDeployment

     # extract svn tags, but only some of them
     git branch -a -l -v | grep remotes/tags |
       grep '^[0-9][0-9]\.' |
       awk '{print substr($1, 14), $2}' |
       while read tag cid; do echo git tag $tag $cid^; done | sh -x

     # extract only the 'Deployment' tree with all its history and tags
     git filter-branch --subdirectory-filter Deployment --prune-empty -- --all

     # review result
     git branch -l -a
     git tag -l
     git log --date-order --pretty=oneline --decorate --graph

     # push to github
     git remote add github git@github.com:mygit/deployment.git
     git push --tags -u github svn

Note that filter-branch can be used with more creative logic to extract only
parts of the tree, for example by renaming or moving files around into a new
layout. All uses of filter-branch will rewrite the git history so it will not
be one-to-one match with the SVN, but it will be materially the same.

Also note that in most cases you very likely should follow this up by removing
all the files in SVN trunk, leaving behind just one "MOVED-TO-GITHUB.txt" file,
with the information where to find the git repository. Of course you should not
delete the entire SVN repository, so checkouts from past history and tags still
works. This is important in case we need to make an urgent bug fix release.
