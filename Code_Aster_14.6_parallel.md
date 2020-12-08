# Parallelization of Code_Aster 14.6

[Open Source](https://qiita.com/tags/オープンソース)[FIVE](https://qiita.com/tags/fem)[OpenCAE](https://qiita.com/tags/opencae)[salome-meca](https://qiita.com/tags/salome-meca)[Code-Aster](https://qiita.com/tags/code-aster)

## 1.First of all

I will explain the procedure of parallelization of Code_Aster.
I proceeded while referring to the following site, but after trial and error, I was finally able to install it.
Some people may be stumbling in the same place, so I'll share an example of a solution.

・[Code_aster 14.6 parallel version with PETSc](https://hitoricae.com/2019/11/10/code_aster-14-4-with-petsc/)
・[Parallelization of Code_Aster 12.6](https://sites.google.com/site/codeastersalomemeca/home/code_asterno-heiretuka/code_asterno-heiretuka-12-6)

## 2 Environment

OS : Ubuntu18.04LTS
Python 3.6.9

## 3 Preparation

Since it will be installed in / opt, change / opt from root to owned by the logged-in user.

```
$ sudo chown username / opt Put your username in     #username
```

Next, install the packages required to build the parallel version of Code_Aster.

```
$ sudo apt install gfortran g++ python-dev python-numpy liblapack-dev libblas-dev tcl tk zlib1g-dev bison flex checkinstall openmpi-bin libx11-dev cmake grace gettext libboost-all-dev swig libsuperlu-dev
```

Next, download the package that needs to be built from the source from the following site and put it in an appropriate directory (~ / software).

| package    | version                               | Download destination                                         |
| :--------- | :------------------------------------ | :----------------------------------------------------------- |
| Code_Aster | aster-full-src-14.6.0-1.noarch.tar.gz | [https://www.code-aster.org](https://www.code-aster.org/)    |
| OpenBLAS   | OpenBLAS-0.2.20.tar.gz                | https://github.com/xianyi/OpenBLAS/                          |
| ScaLAPACK  | scalapack_installer.tgz               | http://www.netlib.org/scalapack/#_scalapack_installer_for_linux |
| Parmetis   | parmetis-4.0.3.tar.gz                 | http://glaros.dtc.umn.edu/gkhome/metis/parmetis/download     |
| Petsc      | petsc-3.9.4.tar.gz                    | https://www.mcs.anl.gov/petsc/download/index.html            |

## 4 OpenBLAS

First, unzip and install.

```shell
cd ~/software
tar xvzf OpenBLAS-0.2.20.tar.gz 
cd OpenBLAS-0.2.20
make NO_AFFINITY=1 USE_OPENMP=1 
make PREFIX=/opt/OpenBLAS install 
```

Add OpenBLAS to the search path of the shared library.

```
$ echo /opt/OpenBLAS/lib | sudo tee -a /etc/ld.so.conf.d/openblas.conf 
$ sudo ldconfig 
```

## 5 Code_Aster with OpenBLAS

In order to parallelize Code_Aster, you need to install the regular version first.
First, unzip it.

```
$ cd ~/software
$ tar xvzf aster-full-src-14.6.0-1.noarch.tar.gz
$ cd aster-full-src-14.6.0
```

Then edit the contents of the setup.cfg file.
Change ```PREFER_COMPILER = GNU``` to ```PREFER_COMPILER = GNU_without_MATH``` and specify the OpenBLAS you just installed for MATHLIB.

```shell
sed -i "s:PREFER_COMPILER\ =\ 'GNU':PREFER_COMPILER\ =\'GNU_without_MATH'\nMATHLIB=\ '/opt/OpenBLAS/lib/libopenblas.a':g" setup.cfg
```

Install it in / opt / aster.

```shell
python3 setup.py install –-prefix=/opt/aster146p
```

You will be asked various questions on the way, but all will be yes.
After the installation is complete, check the operation.

```shell
/opt/aster146p/bin/as_run --vers=14.6 --test forma01a 
```

If there is no error, it is OK.
Create a host file for parallel computing. When forming a cluster with another machine, it seems better to add in the same format.

```shell
echo "$HOSTNAME cpu=$(cat /proc/cpuinfo | grep processor | wc -l)" > /opt/aster146p/etc/codeaster/mpi_hostfile 
```

**TIPS** :  [FAILED] is displayed while running setup.py

In my environment, it seemed to occur when numpy was already installed using pip.
In that case `pip3 uninstall numpy`, delete numpy completely with ⇒ `sudo apt install python-numpy`install numpy with ⇒ install Code_Aster, and try it.

## 6 ScaLAPACK

Unzip and install.

```shell
cd ~/software
tar xvzf scalapack_installer.tgz 
cd scalapack_installer
./setup.py --lapacklib=/opt/OpenBLAS/lib/libopenblas.a --mpicc=mpicc --mpif90=mpif90 --mpiincdir=/usr/lib/x86_64-linux-gnu/openmpi/include --ldflags_c=-fopenmp --ldflags_fc=-fopenmp --prefix=/opt/scalapack
```

**Usually** it will have an error message, such as:

```IOError: [Errno 2] No such file or directory: 'download/./scalapack.tgz'```

Then You need to:

- Download scalapack-2.0.0.tgz from the official website
- Rename and place scalapack.tgz in ~ / software / scalapack_installer / build / ⇒ Run setup.py again.

At the end of the log

```shell
BLACS: error running BLACS test routines xCbtest
BLACS: Command  -np 4 ./xCbtest
stderr:
**************************************
/bin/sh: 1: -np: not found
**************************************
```

Is displayed, but it is successful if the file /opt/scalapack/lib/libscalapack.a is created.



## 7 Parmetis

First, unzip it.

```shell
cd ~ / software
tar xvzf parmetis-4.0.3.tar.gz 
cd parmetis-4.0.3
```

Next, rewrite a part of the file of metis / include / metis.h and change it to compile in 64bit mode.

```shell
sed -i -e 's/#define IDXTYPEWIDTH 32/#define IDXTYPEWIDTH 64/' metis/include/metis.h
```

Install it.

```shell
make config prefix=/opt/parmetis-4.0.3 
make
make install
```

Next, let's check the operation.

```shell
cd Graphs
mpirun -np 4 /opt/parmetis-4.0.3/bin/parmetis rotor.graph 1 6 1 1 6 1
```

If there are no errors, it is successful. On my computer it shows:

```shell
finished reading file: rotor.graph
[ 99617  1324862 24904 24905] [150] [ 0.000] [ 0.000]
[ 53043   786820 13086 13479] [150] [ 0.000] [ 0.000]
[ 28227   423280  6930  7105] [150] [ 0.000] [ 0.000]
[ 15247   229550  3789  3843] [150] [ 0.000] [ 0.000]
[  8304   124178  2044  2121] [150] [ 0.000] [ 0.000]
[  4617    67768  1123  1170] [150] [ 0.000] [ 0.000]
[  2625    36962   643   685] [150] [ 0.000] [ 0.001]
[  1545    20152   360   408] [150] [ 0.000] [ 0.001]
[   944    11180   221   252] [150] [ 0.000] [ 0.002]
[   609     6230   131   161] [150] [ 0.000] [ 0.005]
[   411     3612    78   116] [150] [ 0.000] [ 0.009]
[   347     2916    72   100] [150] [ 0.000] [ 0.009]
nvtxs:        347, cut:    24723, balance: 1.023 
nvtxs:        411, cut:    22942, balance: 1.050 
nvtxs:        609, cut:    22082, balance: 1.044 
nvtxs:        944, cut:    20501, balance: 1.054 
nvtxs:       1545, cut:    19350, balance: 1.052 
nvtxs:       2625, cut:    18147, balance: 1.049 
nvtxs:       4617, cut:    16956, balance: 1.051 
nvtxs:       8304, cut:    15689, balance: 1.049 
nvtxs:      15247, cut:    14632, balance: 1.049 
nvtxs:      28227, cut:    13468, balance: 1.050 
nvtxs:      53043, cut:    12628, balance: 1.048 
nvtxs:      99617, cut:    11288, balance: 1.047 
Final   6-way Cut:  11288 	Balance: 1.047
```



## 8 Scotch

First, move scotch-6.0.4-aster7.tar.gz included in ~ / software / aster-full-src-14.6.0 / SRC to /opt and unzip it.

```
cp ~/software/aster-full-src-14.6.0/SRC/scotch-6.0.4-aster7.tar.gz /opt
cd /opt
tar xvzf scotch-6.0.4-aster7.tar.gz
cd scotch-6.0.4
```

Next, edit Makefile.inc contained in src / as follows.

```makefile
EXE     =
LIB     = .a
OBJ     = .o

MAKE    = make
AR      = ar
ARFLAGS = -ruv
CAT     = cat
CCS     = gcc
CCP     = mpicc
CCD     = gcc
CFLAGS  = -O3 -fPIC -DINTSIZE64 -DCOMMON_FILE_COMPRESS_GZ -DCOMMON_PTHREAD -DCOMMON_RANDOM_FIXED_SEED -DSCOTCH_RENAME -DSCOTCH_RENAME_PARSER -Drestrict=__restrict
CLIBFLAGS   =
LDFLAGS = -fPIC -lz -lm -pthread -lrt
CP      = cp
LEX     = flex -Pscotchyy -olex.yy.c
LN      = ln
MKDIR   = mkdir
MV      = mv
RANLIB  = ranlib
YACC    = bison -pscotchyy -y -b y 
```

Build and check the operation.

```
cd src
make scotch esmumps ptscotch ptesmumps CCD=mpicc
make check
make ptcheck
```

## 9 MUMPS

Again, move mumps-5.1.2-aster6.tar.gz from ~ / software / aster-full-src-14.6.0 / SRC to /opt and unzip it.

```shell
cp ~/software/aster-full-src-14.6.0/SRC/mumps-5.1.2-aster7.tar.gz /opt
cd /opt
tar xvzf mumps-5.1.2-aster6.tar.gz
cd mumps-5.1.2
```

Edit Makefile.inc to suit your environment.
The base file is prepared in the Makefile.inc / directory, so copy it.

```shell
cp Make.inc/Makefile.debian.PAR ./Makefile.inc
```

Modify the library specifications etc. according to the environment.

```makefile
#
# This file is part of MUMPS 5.0.1, changed to be configured by waf scripts
# provided by the Code_Aster team.
#
#Begin orderings

# NOTE that PORD is distributed within MUMPS by default. If you would like to
# use other orderings, you need to obtain the corresponding package and modify
# the variables below accordingly.
# For example, to have Metis available within MUMPS:
#          1/ download Metis and compile it
#          2/ uncomment (suppress # in first column) lines
#             starting with LMETISDIR,  LMETIS
#          3/ add -Dmetis in line ORDERINGSF
#             ORDERINGSF  = -Dpord -Dmetis
#          4/ Compile and install MUMPS
#             make clean; make   (to clean up previous installation)
#
#          Metis/ParMetis and SCOTCH/PT-SCOTCH (ver 5.1 and later) orderings are now available for MUMPS.
#

ISCOTCH    = -I/opt/aster146p/public/metis-5.1.0/include -I/opt/parmetis-4.0.3/include -I/opt/scotch-6.0.4/include
# You have to choose one among the following two lines depending on
# the type of analysis you want to perform. If you want to perform only
# sequential analysis choose the first (remember to add -Dscotch in the ORDERINGSF
# variable below); for both parallel and sequential analysis choose the second 
# line (remember to add -Dptscotch in the ORDERINGSF variable below)

LSCOTCH    = -L/opt/aster146p/public/metis-5.1.0/lib -L/opt/parmetis-4.0.3/lib -L/opt/scotch-6.0.4/lib -L/opt/scalapack-n/lib -Wl,-Bdynamic -lesmumps -lptscotch -lptscotcherr -lptscotcherrexit -lscotch -lscotcherr -lscotcherrexit 
#LSCOTCH    = -L$(SCOTCHDIR)/lib -lptesmumps -lptscotch -lptscotcherr


LPORDDIR = $(topdir)/PORD/lib/
IPORD    = -I$(topdir)/PORD/include/
LPORD    = -L$(LPORDDIR) -lpord

#IMETIS    = # Metis doesn't need include files (Fortran interface avail.)
# You have to choose one among the following two lines depending on
# the type of analysis you want to perform. If you want to perform only
# sequential analysis choose the first (remember to add -Dmetis in the ORDERINGSF
# variable below); for both parallel and sequential analysis choose the second 
# line (remember to add -Dparmetis in the ORDERINGSF variable below)

LMETIS    = -L/opt/aster146p/public/metis-5.1.0/lib -L/opt/parmetis-4.0.3/lib -L/opt/scotch-6.0.4/lib -L/opt/scalapack-n/lib -Wl,-Bdynamic -lparmetis  -Wl,-Bdynamic -lmetis  
#LMETIS    = -L$(LMETISDIR) -lparmetis -lmetis

# The following variables will be used in the compilation process.
# Please note that -Dptscotch and -Dparmetis imply -Dscotch and -Dmetis respectively.
#ORDERINGSF = -Dscotch -Dmetis -Dpord -Dptscotch -Dparmetis
ORDERINGSF  = -Dpord -Dmetis -Dparmetis -Dscotch -Dptscotch
ORDERINGSC  = $(ORDERINGSF)

LORDERINGS = $(LMETIS) $(LPORD) $(LSCOTCH)
IORDERINGSF = $(ISCOTCH)
IORDERINGSC = $(IMETIS) $(IPORD) $(ISCOTCH)

#End orderings
########################################################################
################################################################################

PLAT    =
LIBEXT  = .a
OUTC    = -o 
OUTF    = -o 
RM      = /bin/rm -f
CC      = mpicc
FC      = mpif90
FL      = mpif90
# WARNING: AR must ends with a blank space!
AR      = /usr/bin/ar rcs 
#
RANLIB  = echo

#
INCPAR = -I/opt/aster146p/public/metis-5.1.0/include -I/opt/parmetis-4.0.3/include -I/opt/scotch-6.0.4/include
LIBPAR = 
#
INCSEQ = -I$(topdir)/libseq
LIBSEQ = -L$(topdir)/libseq -lmpiseq

#
LIBBLAS = -L/opt/aster146p/public/metis-5.1.0/lib -L/opt/parmetis-4.0.3/lib -L/opt/scotch-6.0.4/lib -L/opt/scalapack-n/lib -Wl,-Bdynamic -lpthread -lm -lblas -llapack -lscalapack -L/opt/OpenBLAS/lib -lopenblas 
LIBOTHERS =  -L/opt/aster146p/public/metis-5.1.0/lib -L/opt/parmetis-4.0.3/lib -L/opt/scotch-6.0.4/lib -L/opt/scalapack-n/lib -Wl,-Bdynamic -ldl -lutil -lpthread   
#Preprocessor defs for calling Fortran from C (-DAdd_ or -DAdd__ or -DUPPER)
CDEFS   = -D_USE_MPI=1 -DHAVE_MPI=1 -D_USE_OPENMP=1 -DHAVE_METIS_H=1 -D_HAVE_METIS=1 -DHAVE_METIS=1 -DHAVE_PARMETIS_H=1 -D_HAVE_PARMETIS=1 -DHAVE_PARMETIS=1 -DHAVE_STDIO_H=1 -DHAVE_SCOTCH=1 -DAdd_ -Dmetis -Dparmetis

#Begin Optimized options
OPTF    = -O -fPIC -DPORD_INTSIZE64 -fopenmp
OPTL    = -O -Wl,--export-dynamic -fopenmp -L/opt/aster146p/public/metis-5.1.0/lib -L/opt/parmetis-4.0.3/lib -L/opt/scotch-6.0.4/lib -L/opt/scalapack-n/lib -L/usr/lib -L/usr/lib/x86_64-linux-gnu/openmpi/lib -L/opt/aster146p/public/metis-5.1.0/lib -L/opt/parmetis-4.0.3/lib -L/opt/scotch-6.0.4/lib -L/opt/scalapack-n/lib -L/usr//lib -L/usr/lib/x86_64-linux-gnu/openmpi/lib -Lnow -Lrelro -L/opt/aster146p/public/metis-5.1.0/lib -L/opt/parmetis-4.0.3/lib -L/opt/scotch-6.0.4/lib -L/opt/scalapack-n/lib -L/usr//lib -L/usr/lib/x86_64-linux-gnu/openmpi/lib -L/usr/lib/gcc/x86_64-linux-gnu/7 -L/usr/lib/gcc/x86_64-linux-gnu/7/../../../x86_64-linux-gnu -L/usr/lib/gcc/x86_64-linux-gnu/7/../../../../lib -L/lib/x86_64-linux-gnu -L/lib/../lib -L/usr/lib/x86_64-linux-gnu -L/usr/lib/../lib -L/usr/lib/gcc/x86_64-linux-gnu/7/../../.. -lmpi_usempif08 -lmpi_usempi_ignore_tkr -lmpi_mpifh -lmpi -lgfortran -lquadmath -lpthread -L/opt/aster146p/public/metis-5.1.0/lib -L/opt/parmetis-4.0.3/lib -L/opt/scotch-6.0.4/lib -L/opt/scalapack-n/lib -L/usr//lib -L/usr/lib/x86_64-linux-gnu/openmpi/lib
OPTC    = -O -fPIC -fopenmp
#End Optimized options

INCS = $(INCPAR)
LIBS = $(LIBPAR)
LIBSEQNEEDED = 
```

Build and check the operation.

```shell
make all
cd examples
mpirun -np 4 ./ssimpletest < input_simpletest_real
```

If there is no error, it is OK.

## 10 Petsc

First, unzip it with / opt.

```shell
cd /opt
tar xvzf ~/software/petsc-3.9.4.tar.gz
cd petsc-3.9.4
```

Then open the metis.py file located at /opt/petsc-3.9.4/config/BuildSystem/config/packages and comment out lines 43-48.
It will be the following part.

metis.py

```python
def configureLibrary(self):
    config.package.Package.configureLibrary(self)
    oldFlags = self.compilers.CPPFLAGS
    self.compilers.CPPFLAGS += ' '+self.headers.toString(self.include)
#    if not self.checkCompile('#include "metis.h"', '#if (IDXTYPEWIDTH != '+ str(self.getDefaultIndexSize())+')\n#error incompatible IDXTYPEWIDTH\n#endif'):
#      if self.defaultIndexSize == 64:
#        msg= '--with-64-bit-indices option requires a metis build with IDXTYPEWIDTH=64.\n'
#      else:
#        msg= 'IDXTYPEWIDTH=64 metis build appears to be specified for a default 32-bit-indices build of PETSc.\n'
#      raise RuntimeError('Metis specified is incompatible!\n'+msg+'Suggest using --download-metis for a compatible metis')

    self.compilers.CPPFLAGS = oldFlags
    return
```

In addition, register the OpenMPI library in LD_LIBRARY_PATH, and then execute configure.

```shell
export LD_LIBRARY_PATH=/usr/lib/x86_64-linux-gnu/openmpi/lib/:$LD_LIBRARY_PATH
/configure --with-debugging=0 COPTFLAGS=-O CXXOPTFLAGS=-O FOPTFLAGS=-O --with-shared-libraries=0 --with-scalapack-dir=/opt/scalapack --PETSC_ARCH=linux-metis-mumps --with-metis-dir=/opt/aster146p/public/metis-5.1.0 --with-parmetis-dir=/opt/parmetis-4.0.3 --with-ptscotch-dir=/opt/scotch-6.0.4 --LIBS="-lgomp" --with-mumps-dir=/opt/mumps-5.1.2 -with-x=0 --with-blas-lapack-lib=[/opt/OpenBLAS/lib/libopenblas.a] --download-hypre=yes --download-ml=yes
```

**TIPS**

1. If the hypre download doesn't work

- It will tell you the URL of the download destination github, but it seems that the repository does not exist ...
- Since it is registered in the launchpad of Ubuntu, you can download it from there (file name: hypre_2.14.0.orig.tar.gz).
  　[hypre 2.14.0-5build1 source package in Ubuntu](https://launchpad.net/ubuntu/+source/hypre/2.14.0-5build1)
- After downloading, put it in / opt.
- After that, change `--download-hypre=yes`the part to `--download-hypre=/opt/hypre_2.14.0.orig.tar.gz`and try configure again.

2. If the ml download doesn't work

- This can be downloaded from the specified URL. Download it and place it in / opt. (File name: petsc-pkg-ml-e5040d11aa07.zip)
  [bitbucket pkg-ml](https://bitbucket.org/petsc/pkg-ml/downloads/?tab=downloads)
- Similar to hypre , change `--download-ml=yes`the part to `--download-ml=/opt/petsc-pkg-ml-e5040d11aa07.zip`and run configure again.

After successfully configure, run make.

```shell
make PETSC_DIR=/opt/petsc-3.9.4 PETSC_ARCH=linux-metis-mumps all
make PETSC_DIR=/opt/petsc-3.9.4 PETSC_ARCH=linux-metis-mumps check
```

## 11 Side by side Code_Aster

There is a parallel version Code_Aster source file in the Code_Aster source file, so unzip it first.

```shell
cd ~/software/aster-full-src-14.6.0/SRC
tar xfvz aster-14.6.0.tgz
cd aster-14.6.0
```

Comment out lines 362-364 of waftools / mathematicals.py in the above directory.
It will be the following part.

mathematics.py

```python
# program testing a blacs call, output is 0 and 1
blacs_fragment = r"""
program test_blacs
    integer iam, nprocs
#    call blacs_pinfo (iam, nprocs)
#    print *,iam
#    print *,nprocs
end program test_blacs
"""
```

Next, create Ubuntu_gnu_mpi.py and Ubuntu_gnu.py and place them in your current directory (~ / software / aster-full-src-14.6.0 / SRC / aster-14.6.0).



```python
# encoding: utf-8

"""
Fichier de configuration WAF pour version parallﾃｨle sur Ubuntu 13.6 :
- Compilateur : GNU
- MPI         : systﾃｨme (OpenMPI, Ubuntu 13.6)
- BLAS        : OpenBLAS
- Scalapack   : systﾃｨme (Ubuntu 13.6)
- PETSc       : 
"""

import Ubuntu_gnu

def configure(self):
    opts = self.options
    Ubuntu_gnu.configure(self)

    self.env.prepend_value('LIBPATH', [
        '/opt/petsc-3.9.4/linux-metis-mumps/lib',
        '/opt/parmetis-4.0.3/lib',
        '/opt/mumps-5.1.2/lib',])

    self.env.prepend_value('INCLUDES', [
        '/opt/petsc-3.9.4/linux-metis-mumps/include',
        '/opt/petsc-3.9.4/include',
        '/usr/include/superlu',
        '/opt/parmetis-4.0.3/include',
        '/opt/mumps-5.1.2/include',])

    self.env.append_value('LIB', ('X11',))

    opts.parallel = True

    opts.enable_mumps  = True
    opts.mumps_version = '5.1.2'
    opts.mumps_libs = 'dmumps zmumps smumps cmumps mumps_common pord metis scalapack openblas esmumps scotch scotcherr'
#    opts.embed_mumps = True

    opts.enable_petsc = True
    opts.petsc_libs='petsc HYPRE ml'
#    opts.petsc_libs='petsc'
#    opts.embed_petsc = True

#    opts.enable_parmetis  = True
    self.env.append_value('LIB_METIS', ('parmetis'))
    self.env.append_value('LIB_SCOTCH', ('ptscotch','ptscotcherr','ptscotcherrexit','ptesmumps'))
```



**Ubuntu_gnu.py**

```python
# encoding: utf-8

"""
Fichier de configuration WAF pour version sﾃｩquentielle sur Ubuntu 13.6 :
- Compilateur : GNU
- BLAS        : OpenBLAS
"""
import os

def configure(self):
    opts = self.options

    # mfront path
#    self.env.TFELHOME = '/opt/tfel-3.2.0'

    self.env.append_value('LIBPATH', [
        '/opt/aster146p/public/hdf5-1.10.3/lib',
        '/opt/aster146p/public/med-4.0.0/lib',
        '/opt/aster146p/public/metis-5.1.0/lib',
        '/opt/scotch-6.0.4/lib',
        '/opt/OpenBLAS/lib',
        '/opt/scalapack/lib',])
#        '/opt/tfel-3.2.0/lib',

    self.env.append_value('INCLUDES', [
        '/opt/aster146p/public/hdf5-1.10.3/include',
        '/opt/aster146p/public/med-4.0.0/include',
        '/opt/aster146p/public/metis-5.1.0/include',
        '/opt/scotch-6.0.4/include',
        '/opt/OpenBLAS/include',
        '/opt/scalapack/include',])
#        '/opt/tfel-3.2.0/include',

    opts.'openblas superlu'=maths_libs  
#    opts.embed_math = True

    opts.enable_hdf5 = True
    opts.hdf5_libs  = 'hdf5 z'
#    opts.embed_hdf5 = True

    opts.enable_med = True
    opts.med_libs  = 'med stdc++'
#    opts.embed_med  = True

    opts.enable_mfront = False

    opts.enable_scotch = True
#    opts.embed_scotch  = True

    opts.enable_homard = True
#    opts.embed_aster    = True
#    opts.embed_fermetur = True

    # add paths for external programs
#    os.environ['METISDIR'] = '/opt/aster146p/public/metis-5.1.0'
#    os.environ['GMSH_BIN_DIR'] = '/opt/aster146p/public/gmsh-3.0.6-Linux/bin'
    os.environ['HOMARD_ASTER_ROOT_DIR'] = '/opt/aster146p/public/homard-11.12'

    opts.with_prog_metis = True
#    opts.with_prog_gmsh = True
    # salome: only required by few testcases
    # europlexus: not available on all platforms
#    opts.with_prog_miss3d = True
    opts.with_prog_homard = True
#    opts.with_prog_ecrevisse = True
    opts.with_prog_xmgrace = True
```





I will install it when I am ready.

```shell
export ASTER_ROOT=/opt/aster146p
export PYTHONPATH=/$ASTER_ROOT/lib/python3.6/site-packages/:$PYTHONPATH
./waf configure --use-config-dir=$ASTER_ROOT/14.6/share/aster --use-config=Ubuntu_gnu_mpi --prefix=$ASTER_ROOT/PAR14.6MUPT
./waf install -p --jobs=1
```

When you're done, register it with the name 14.6MUPT so that you can use the parallel version of Code_Aster with ASTK.
There is a file called aster in / opt / aster / etc / codeaster /, so add the following to the last line of it.

```makefile
# Code_Aster versions
# versions can be absolute paths or relative to ASTER_ROOT
# examples : NEW11, /usr/lib/codeaster/NEW11

# default version (overridden by --vers option)
default_vers : stable

# available versions
# DO NOT EDIT FOLLOWING LINE !
#?vers : VVV?
vers : stable:/opt/aster146p/14.6/share/aster
vers : 14.6MUPT:/opt/aster146p/PAR14.6MUPT/share/aster
```

That's all.
When you start ASTK, 14.6 MUPT is newly added and can be selected in the Version tab.
[![astk3.png](https://qiita-user-contents.imgix.net/https%3A%2F%2Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%2F0%2F439095%2Fda95ba00-57a1-dc6d-935e-dd89b012f5d6.png?ixlib=rb-1.2.2&auto=format&gif-q=60&q=75&s=75b9b9e8b34c6fcf356caeb8787e2df5)](https://qiita-user-contents.imgix.net/https%3A%2F%2Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%2F0%2F439095%2Fda95ba00-57a1-dc6d-935e-dd89b012f5d6.png?ixlib=rb-1.2.2&auto=format&gif-q=60&q=75&s=75b9b9e8b34c6fcf356caeb8787e2df5)

Thank you for your hard work!
Thank you to those who have disclosed the information.