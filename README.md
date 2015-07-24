# DMWM Developer Tutorials

To generate the documentation, source sphinx environment and do:

    make html

You will find the documentation in `.build/html/index.html`.

To generate twiki pages, use:

    make twiki

Where the twikis will be in `.build/twiki/`.

For more available options, check `make help`.

The official location for the generated documentation is https://cern.ch/cms-http-group/tutorials/.

The https://twiki.cern.ch/twiki/bin/view/CMS/DMWMTutorials page is a mirror 
only to allow people to search from the main CERN twiki system.

Typically, the official deployment is:

    (set -e;
     git clone git://github.com/dmwm/tutorials.git;
     cd tutorials/;
     source /build/dmwmbld/srv/state/dmwmbld/builds/comp_gcc481/w/slc6_amd64_gcc481/cms/wmcore-devtools/1.2-comp4/etc/profile.d/init.sh;
     make html;
     make twiki;
     kinit cmsweb;
     rsync -rvu ./.build/html/ ./.build/twiki /afs/cern.ch/user/c/cmsweb/www/tutorials/)
