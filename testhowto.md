### Clone example 1: numpy builtin test suite
As user@justus-login01.rz.uni-ulm.de (use ssh -A)

#### let's have a look at the available SW stack
```
module avail
# ...
```

#### load numpy module
`module load numlib/python_numpy/1.9.1`

#### see what modules are loaded (numpy+dependencies)
```
module list
# Currently Loaded Modulefiles:
#  1) numlib/mkl/11.1.4           2) numlib/python_numpy/1.9.1
```
#### find important paths:
```
module show numlib/mkl/11.1.4
module show numlib/python_numpy/1.9.1
```

#### Check dirs...
```
ls -lah $MKLROOT
lrwxrwxrwx 1 ul_l_cmosch adm-sw 14 Nov  2  2014 /opt/bwhpc/common/compiler/intel/compxe.2013.sp1.4.211/mkl -> composerxe/mkl
ls -lah $MKLROOT/
# ...

ls -lah $PYTHONPATH
```
#### rsync the mkl comp/lib dependency
```
rsync -avzPK $MKLROOT/ centos@134.60.51.52:mkl+numpy/mkl
rsync -avzPK /opt/bwhpc/common/compiler/intel/compxe.2013.sp1.4.211/lib/intel64/ centos@134.60.51.52:mkl+numpy/mkl/stdlib/intel64
```
#### rsync the python/numpy dependency
`rsync -avzPK $PYTHONPATH centos@134.60.51.52:mkl+numpy/numpy`

#### NOW switch to the virtual host (bwCloud)
```
ssh centos@134.60.51.52
sudo yum install python python-nose
export PREFIX=~/mkl+numpy
export LD_LIBRARY_PATH=$PREFIX/mkl/lib/intel64:$PREFIX/mkl/stdlib/intel64
export PYTHONPATH=$PREFIX/numpy

/bin/python --version
# 2.7.5

/bin/python
# import numpy
# numpy.test()
# nose version 1.3.0
# python version 2.7.6
# numpy version 1.9.1
# ...
# Ran 5222 tests in 24.008s / ~27sec on single core JUSTUS
#
# OK (KNOWNFAIL=5, SKIP=14)
# <nose.result.TextTestResult run=5222 errors=0 failures=0>
# >>>
```

## Clone example 2: vasp benchmark

As user@justus-login01.rz.uni-ulm.de (use ssh -A)

# load vasp module
module load chem/vasp/5.3.3.4

# show env modifications
module show chem/vasp/5.3.3.4

# TODO...

```
