DMWM build process
------------------

::

  # cf. https://twiki.cern.ch/twiki/bin/view/CMS/DMWMBuilds for current tags
  mkdir -p /build/$USER/comp
  cd /build/$USER/comp
  cvs -Q co -r HG1205f -d CMSDIST-1205 CMSDIST
  cvs -Q co -r V00-20-27 PKGTOOLS

  PKGTOOLS/cmsBuild -c CMSDIST-1205 \
    --repository comp.pre --arch slc5_amd64_gcc461 \
    --builders=8 -j 5 --work-dir w build reqmgr

  PKGTOOLS/cmsBuild -c CMSDIST-1205 \
    --repository comp.pre --arch slc5_amd64_gcc461 \
    --builders=8 -j 5 --work-dir w upload reqmgr
