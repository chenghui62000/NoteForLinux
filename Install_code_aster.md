# How to install Code_Aster

**Note**: Most of the time, Code_Aster is installed with the series way on a computer. 

---

## Install series version
This is a note of the work on linux Mint 19.3
Ref1: ([code_Aster official website](https://www.code-aster.org/spip.php?article272))
### 1. Download the package
Download from the [Office website](https://code-aster.org/V2/spip.php?article272)
Unzip the package 
```
tar -xvf aster-xxxxx.tar.gz
```
**NOTE**: according to my tests, Code_Aster 14.4 is quite unstable for Non-linear dynamic simulations. Thus, I recommond Code_Aster 14.2.

### 2. Install prerequiesites

``` shell
sudo apt-get install gcc g++ gfortran cmake python3 python3-dev python3-numpy tk tcl bison flex liblapack-dev libblas-dev libopenblas-dev libboost-python-dev libboost-numpy-dev zlib1g-dev nedit ddd
```

### 3. Install the main program
   Then Install Code_Aster:	
   ``` shell
   sudo python setup.py install --prefix=/opt/aster
   ```
### 4. Great alias command
 Add the following to the ~/.bashrc, so that you can enter the Code_Aster enviroment and run simulations easily. 

   ``` shell
   alias aster='source /opt/aster/etc/codeaster/profile.sh'
   ```

**Now, the installation for the series version is finished.**

---

## Install parallel version
This is a note of the work to build parallel version on linux Mint 19.3

Ref1: ([A Blog by a Jananese](https://hitoricae.com/2020/05/16/installation-code_aster14-4-to-xubuntu20-04/))

Ref2:([code_aster 14.4 parallel version with PETSc](https://hitoricae.com/2019/11/10/code_aster-14-4-with-petsc/))

Ref3: ([parallel version based on CA14.2](https://code-aster.it/2019/01/12/code_aster-14-2-in-parallelo-su-ubuntu-bionic/))


1. Install prerequisites
``` shell
sudo apt-get install gcc g++ gfortran cmake python3 python3-dev python3-numpy tk tcl bison flex liblapack-dev libblas-dev libopenblas-dev libboost-python-dev libboost-numpy-dev zlib1g-dev nedit geany vim ddd

sudo apt-get install checkinstall openmpi-bin libx11-dev grace gettext libboost-all-dev swig libsuperlu-dev
```

2. Download and install OpenBLAS 
   Official website for [OpenBLAS](https://www.openblas.net/)
   
   ```shell
   tar xfvz OpenBLAS-0.2.20.tar.gz
   cd OpenBLAS-0.2.20
   make NO_AFFINITY=1 USE_OPENMP=1
   make PREFIX=/opt/OpenBLAS install
   echo /opt/OpenBLAS/lib | sudo tee -a /etc/ld.so.conf.d/openblas.conf
   sudo ldconfig
   ```
   
3. Configure the Code_Aster install setting
   
   Change the compiler in file "setup.cfg", line 37:
   
   ```
   PREFER_COMPILER = 'GNU_without_MATH'
   MATHLIB='/opt/OpenBLAS/lib/libopenblas.a'
   ```
   Then Install Code_Aster:	
   
   ``` 
   python3 setup.py install --prefix=/opt/aster144
   ```
--- 
Not finish yet



4. After the build complete, to make host file for parallel calculation. (Write the description to the mpi_hostfile)
   
   ```
    echo "$HOSTNAME cpu=$(cat /proc/cpuinfo | grep processor | wc -l)" > /opt/aster/etc/codeaster/mpi_hostfile
    ```
5.  ScaLAPACK  [offical website](http://www.netlib.org/scalapack/)

   ```
   tar xfvz scalapack_installer.tgz
   cd scalapack_installer
   ./setup.py --lapacklib=/opt/OpenBLAS/lib/libopenblas.a --mpicc=mpicc --mpif90=mpif90 --mpiincdir=/usr/lib/x86_64-linux-gnu/openmpi/include --ldflags_c=-fopenmp --ldflags_fc=-fopenmp --prefix=/opt/scalapack-n

   ```

