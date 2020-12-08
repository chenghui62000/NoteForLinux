
## Install parallel version
This is a note of the work to build parallel version on linux Mint 19.3

Ref1: ([A Blog by a Jananese](https://hitoricae.com/2020/05/16/installation-code_aster14-4-to-xubuntu20-04/))

Ref2:([code_aster 14.4 parallel version with PETSc](https://hitoricae.com/2019/11/10/code_aster-14-4-with-petsc/))

### Method 1
https://hitoricae.com/2020/10/31/code_aster-14-6-parallel-version-with-petsc/

1. Install prerequisites
``` shell
sudo apt-get install  gfortran g++ python-dev python-numpy liblapack-dev libblas-dev tcl tk zlib1g-dev bison flex checkinstall openmpi-bin libx11-dev cmake grace gettext libboost-all-dev swig libsuperlu-dev
```

2. [OpenBlas ](https://www.openblas.net/)

Must be version 0.2.20

```shell
tar xfvz OpenBLAS-0.2.20.tar.gz
cd OpenBLAS-0.2.20
make NO_AFFINITY=1 USE_OPENMP=1
sudo make PREFIX=/opt/OpenBLAS install
echo /opt/OpenBLAS/lib | sudo tee -a /etc/ld.so.conf.d/openblas.conf
sudo ldconfig

```
3. [Code_Aster](https://www.code-aster.org/V2/spip.php?article272 ) with OpenBLAS
```shell
cd aster-full-src-14.6.0
sed -i "s:PREFER_COMPILER\ =\ 'GNU':PREFER_COMPILER\ =\'GNU_without_MATH'\nMATHLIB=\ '/opt/OpenBLAS/lib/libopenblas.a':g" setup.cfg
```
And:
```shell
sudo python3 setup.py install
```

After successfully install the code aster, Then write to the mpi_hostfile:
```shell
echo "$HOSTNAME cpu=$(cat /proc/cpuinfo | grep processor | wc -l)" > /opt/aster/etc/codeaster/mpi_hostfile
```


4. [ScaLAPACK](http://www.netlib.org/scalapack/#_scalapack_version_2_1_0)

You need to manually download ScaLAPACK, version 2.0.0, and put it
into the current directory: **/scalapack_installer/build/download

```shell
tar xfvz scalapack_installer.tgz
cd scalapack_installer
sudo ./setup.py --lapacklib=/opt/OpenBLAS/lib/libopenblas.a --mpicc=mpicc --mpif90=mpif90 --mpiincdir=/usr/lib/x86_64-linux-gnu/openmpi/include --ldflags_c=-fopenmp --ldflags_fc=-fopenmp --prefix=/opt/scalapack-n
```

5. [Parmetis](http://glaros.dtc.umn.edu/gkhome/metis/parmetis/download)

```
cd parmetis
make config prefix=${ASTER_PUBLIC}/parmetis-${PARMETIS_VER}
make -j
```
Next, checking.
``` shell
cd Graphs
mpirun -np 8 ptest rotor.graph rotor.graph.xyz
```
When on error is reported,
```
cd ..
make install
```

**#STOP HERE#**




















### Method 2
Ref3: ([parallel version based on CA14.2](https://code-aster.it/2019/01/12/code_aster-14-2-in-parallelo-su-ubuntu-bionic/))

1. Install prerequisites
```
sudo apt-get install -y curl mercurial gfortran g++ python-dev python-numpy liblapack-dev libblas-dev tcl tk zlib1g-dev bison flex checkinstall openmpi-bin cmake hdf5-tools swig grace gettext libboost-dev libboost-python-dev libopenmpi-dev tkpng libatlas-base-dev libblacs-mpi-dev libsuperlu-dev

```



2. Set up environment:

**NOTE1:** Be careful with the version number! Confirm with your own computer!

**NOTE2:** The DEV_DIR and INSTALL_DIR can be any location that you have permission to access.

``` shell
export ASTER_VER=14.6
export DEV_DIR=~/dev
export INSTALL_DIR=~/install
export HDF5_VER=1.10.3
export MED_VER=4.0.0
export METIS_VER=5.1.0
export PARMETIS_VER=4.0.3
export SCOTCH_VER=6.0.4
export MUMPS_VER=5.1.2
export MFRONT_VER=3.1.1
export PETSC_VER=3.8.2
export SCALAPACK_VER=2.0.2
export ASTER_ROOT=$INSTALL_DIR/aster
export ASTER_PUBLIC=$ASTER_ROOT/public
export ASTER_FULLSRC_DIR=$DEV_DIR/aster-full-src-${ASTER_VER}.0

mkdir $INSTALL_DIR
mkdir -p $DEV_DIR/aster-prerequisites
mkdir -p $DEV_DIR/codeaster
```
3. download and compile the sequential version of Code_Aster:
``` shell
curl -Sl https://www.code-aster.org/FICHIERS/aster-full-src-$ASTER_VER.0-1.noarch.tar.gz | tar -xzC $DEV_DIR

cd $ASTER_FULLSRC_DIR
python3 setup.py --prefix $INSTALL_DIR/aster
```

4. install ptscotch
``` shell
cd $DEV_DIR/aster-prerequisites
mkdir ptscotch

tar xf $ASTER_FULLSRC_DIR/SRC/scotch-${SCOTCH_VER}-aster5.tar.gz -C ptscotch --strip-components 1
```
**NOTE** change the file: ```$DEV_DIR/aster-prerequisites/ptscotch/src/Makefile.inc```, change the line 12 to:
```
CFLAGS		= -O3 -fPIC -DCOMMON_FILE_COMPRESS_GZ -DCOMMON_PTHREAD -DCOMMON_RANDOM_FIXED_SEED -DSCOTCH_RENAME -DSCOTCH_RENAME_PARSER -Drestrict=__restrict
```
Then 
```shell
cd ptscotch/src
make scotch esmumps ptscotch ptesmumps CCD=mpicc
mkdir ${ASTER_PUBLIC}/ptscotch-${SCOTCH_VER}
make install prefix=${ASTER_PUBLIC}/ptscotch-${SCOTCH_VER}
```
5. parmetis
```shell
cd parmetis
make config prefix=${ASTER_PUBLIC}/parmetis-${PARMETIS_VER}
make -j
make install
```

6. scalapack

new version is 2.1.0
```shell
mkdir scalapack

curl -Sl http://www.netlib.org/scalapack/scalapack-${SCALAPACK_VER}.tgz | tar -xzC $DEV_DIR/aster-prerequisites/scalapack --strip-components 1

cd scalapack && mkdir build && cd build

cmake -DCMAKE_INSTALL_PREFIX=${ASTER_PUBLIC}/scalapack-${SCALAPACK_VER} -DBUILD_SHARED_LIBS=ON -DBUILD_STATIC_LIBS=ON ..

make -j && make install
cd ${ASTER_PUBLIC}/scalapack-${SCALAPACK_VER}/lib
cp libscalapack.so libblacs.so

```

7.Mumps 

new version is 5.3.5
but code_aster14.6 uses 5.1.2 version
``` shell
cd $DEV_DIR/aster-prerequisites
mkdir mumps
tar xf $ASTER_FULLSRC_DIR/SRC/mumps-${MUMPS_VER}-aster5.tar.gz -C mumps --strip-components 1
cd mumps

INCLUDES="${ASTER_PUBLIC}/metis-${METIS_VER}/include ${ASTER_PUBLIC}/parmetis-${PARMETIS_VER}/include ${ASTER_PUBLIC}/ptscotch-${SCOTCH_VER}/include" LIBPATH="${ASTER_PUBLIC}/metis-${METIS_VER}/lib ${ASTER_PUBLIC}/parmetis-${PARMETIS_VER}/lib ${ASTER_PUBLIC}/ptscotch-${SCOTCH_VER}/lib ${ASTER_PUBLIC}/scalapack-${SCALAPACK_VER}/lib" ./waf configure --prefix=${ASTER_PUBLIC}/mumps-${MUMPS_VER}_mpi --install-tests --enable-mpi

./waf build --jobs=1
./waf install --jobs=1
```

8. install valgrind first

```shell
sudo apt-get install valgrind
```

9. petsc

**NOTE** you might meet some configure error! DONT be afraid! fellow the error message and download and install some package!

new version is 3.14.0

```shell
cd $DEV_DIR/aster-prerequisites
mkdir petsc

curl -Sl https://bitbucket.org/petsc/petsc/get/v${PETSC_VER}.tar.gz | tar --strip-components 1 -xzC $DEV_DIR/aster-prerequisites/petsc

cd petsc

./configure --COPTFLAGS="-O2" --CXXOPTFLAGS="-O2"  --FOPTFLAGS="-O2"  --with-debugging=0 --with-shared-libraries=1  --with-scalapack-dir=${ASTER_PUBLIC}/scalapack-${SCALAPACK_VER}  --with-mumps-dir=${ASTER_PUBLIC}/mumps-${MUMPS_VER}_mpi  --with-metis-dir=${ASTER_PUBLIC}/metis-${METIS_VER}  --with-ptscotch-dir=${ASTER_PUBLIC}/ptscotch-${SCOTCH_VER} --download-hypre --download-ml  --LIBS="-lgomp" --prefix=${ASTER_PUBLIC}/petsc-${PETSC_VER}

make all && make install

cp ./arch-linux2-c-opt/lib/libpetsc.so ${ASTER_PUBLIC}/petsc-${PETSC_VER}/lib/
```

10 compile Code_Aster in parallel:

sudo apt install libsuperlu-dev

```shell
cd $DEV_DIR/
source ${ASTER_ROOT}/${ASTER_VER}/share/aster/profile_mfront.sh
export METISDIR=${ASTER_PUBLIC}/metis-${METIS_VER}
export TFELHOME=${ASTER_PUBLIC}/tfel-${MFRONT_VER}
export GMSH_BIN_DIR=${ASTER_PUBLIC}/gmsh-3.0.6-Linux64/bin
export HOMARD_ASTER_ROOT_DIR=${ASTER_PUBLIC}/homard-11.12
export PYTHONPATH=/$ASTER_ROOT/lib/python3.6/site-packages/:$PYTHONPATH
mkdir aster

tar xf $ASTER_FULLSRC_DIR/SRC/aster-${ASTER_VER}.0.tgz -C aster --strip-components 1

cd aster

cat >> $DEV_DIR/aster/wafcfg/Ubuntu_mpi.py << 'EOF'
def configure(self):
    self.env.append_value('LIB_METIS', ('parmetis'))
    self.env.append_value('LIB_SCOTCH', ('ptscotch', 'ptscotcherr', 'ptscotcherrexit'))
EOF
```
I failed in the following step
```
INCLUDES="${ASTER_PUBLIC}/hdf5-${HDF5_VER}/include ${ASTER_PUBLIC}/med-${MED_VER}/include ${ASTER_PUBLIC}/metis-${METIS_VER}/include ${ASTER_PUBLIC}/parmetis-${PARMETIS_VER}/include ${ASTER_PUBLIC}/ptscotch-${SCOTCH_VER}/include ${ASTER_PUBLIC}/mumps-${MUMPS_VER}_mpi/include ${ASTER_PUBLIC}/petsc-${PETSC_VER}/include ${ASTER_PUBLIC}/tfel-${MFRONT_VER}/include" LIBPATH="${ASTER_PUBLIC}/hdf5-${HDF5_VER}/lib ${ASTER_PUBLIC}/med-${MED_VER}/lib ${ASTER_PUBLIC}/metis-${METIS_VER}/lib ${ASTER_PUBLIC}/parmetis-${PARMETIS_VER}/lib ${ASTER_PUBLIC}/ptscotch-${SCOTCH_VER}/lib ${ASTER_PUBLIC}/scalapack-${SCALAPACK_VER}/lib ${ASTER_PUBLIC}/mumps-${MUMPS_VER}_mpi/lib ${ASTER_PUBLIC}/petsc-${PETSC_VER}/lib ${ASTER_PUBLIC}/tfel-${MFRONT_VER}/lib" ./waf configure --use-config=Ubuntu_mpi --prefix=${ASTER_ROOT}/${ASTER_VER}_mpi --install-tests --enable-mpi

./waf build && ./waf install

```

We need to register the new version of Code_Aster to make it available and the current machine as an MPI node, also we need to add the Code_Aster binaries in the PATH (possibly add them to ~ / .bashrc):
```shell
echo "vers : testing_mpi:${ASTER_ROOT}/${ASTER_VER}_mpi/share/aster" >> ${ASTER_ROOT}/etc/codeaster/aster

echo "vers : ${ASTER_VER}_mpi:${ASTER_ROOT}/${ASTER_VER}_mpi/share/aster" >> ${ASTER_ROOT}/etc/codeaster/aster

echo "localhost" > ${ASTER_ROOT}/etc/codeaster/mpi_hostfile
export PATH=$ASTER_ROOT/bin:$PATH
as_run --info
```





