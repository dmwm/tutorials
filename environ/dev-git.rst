Working with git for CMS DMWM projects
--------------------------------------

Within the distributed nature of GIT, every repository checkout is
a full copy of the repository content and its history.
We don't have any central server and all the development
can be done offline.

On this serverless model, however, we must decide what's the authoritative
(aka "official") copy of the repository from where people would clone from and to
where they contribute back. For DMWM projects, the official repositories are hosted
in the `'dmwm' account in github <https://github.com/dmwm>`_. For CMSSW projects,
it is the `'cms-sw' account in github <https://github.com/cms-sw>`_. In some cases,
copies of some of the repositories may be made every once in a while to the central
`GIT service at CERN <https://git.cern.ch/web/>`_ as a protection against disasters.

Besides hosting the official repositories copy, we also use github services
to exchange/review the changes we want to contribute to the official repositories.
This is done through github
`pull requests <https://help.github.com/articles/using-pull-requests>`_.
Some projects may also use github for issue tracking and the project
documentation (wiki).

In order to send a pull request, you must first have created your own
github account and set the SSH keys you'll use when pushing your changes
to it. In the instructions below, we call your github account 'mygit'.


Doing changes and sending a pull request
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

1. Locate the code repository you want to contribute. In this example
   we will use the
   `'deployment' repository from 'dmwm' user <https://github.com/dmwm/deployment>`_.
   Then fork the repository to your own account by clicking in the 'fork'
   button on the top right corner of this project area in github. As
   a result, you'll get a copy of the repository under your 'mygit' account
   in github.

2. Clone the repository from github/mygit the to your local development area,
   and set up 'upstream' git remote for the original repo, and a tracking branch
   'upstream' for the upstream's master branch. ::

       cd My/Dev/Area
       git clone git@github.com:mygit/deployment.git
       cd deployment

       git remote add upstream git://github.com/dmwm/deployment.git
       git fetch upstream
       git branch --track upmaster remotes/upstream/master

3. To start a new development, first update the 'upmaster' branch to
   current tip of the upstream repository. ::

       git checkout upmaster
       git pull

4. Create a new feature branch for your development. We'll call this
   'hg1305-reqmgr-update', obviously you should pick a name appropriate
   for your changes. ::

       git checkout -b hg1305-reqmgr-update

5. Modify the code and commit. If you like using 'stg', you can and
   should use it here to manage your patch stack. In that case, at the
   beginning say 'stg init' to initialise your branch for stg, then at
   the end when you are ready to submit, do 'stg commit -a' to commit
   the patches on your branch. The following assumes raw git use. ::

       vi reqmgr/deploy
       git add reqmgr/deploy
       git commit -m "Incredibly awesome changes."

6. Merge with upstream, in case there were changes in the mean time.
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

       git checkout hg1305-reqmgr-update
       git merge upmaster

7. Push the changes to your github account. The 'origin' is where you did
   the initial clone, i.e. the 'deployment' repository in *your* github
   account 'mygit'. You should push both your 'master' (see above why) and all the
   feature branches you worked on. This is your last chance to review the
   changes before publishing them. ::

       # review history
       git log --date-order --pretty=oneline --decorate --graph

       # review with full details, including patches
       git log -p

       git push origin master hg1305-reqmgr-update

8. At github web site, select your account's 'deployment' repository, and
   the 'hg1305-reqmgr-update' branch, and follow the pull request recipe
   to create a pull request for the 'deployment' project in 'dmwm'.
   See `<http://help.github.com/send-pull-requests/>`_.

   The dmwm managers will then review your changes and merge them to the target
   branch of dmwm/deployment (usually it is the 'master') you selected when
   filling up the pull request.

   Done! You got your changes contributed to the target project!


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

       git merge joebloggs/hg1305-reqmgr-update
       git push origin master

6. Note that other users go back into
   `Doing changes and sending a pull request`_, step 3 to pull
   the changes you have just committed. Also note that git has no single
   master, so in fact anyone can execute the steps in
   `Applying a pull request after a review`_
   and push to their own repo, and anyone else can merge those changes. This
   means work can proceed from any repository. In all likelihood we will try
   to keep repositories under 'dmwm' user with current state of the art at all
   times, with multiple committers with the necessary rights.


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
