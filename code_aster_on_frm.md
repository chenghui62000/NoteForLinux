
The compilation is a little bit tricky. Follow the instructions in order please:

1. Extract Code_asters version 14.6 tar.gz file to your project folder and create another empty folder for binary files in your project folder.

```
$mkdir -p /cluster/home/huicheng/aster/public
```
2. 
Load the following modules in the same order.

```
$module load Boost/1.68.0-intel-2018b-Python-3.6.6
$module load CMake/3.12.1
$module load intel/2020a
```

When you type module list command you should see 22 modules loaded in total.

3. compile hdf and med manually
Go to SRC folder in the code_aster folder and extract hdf5-1.10.3  and med-4.0.0 folder. 

    * 3.1

And compile the hdf5 library first by using GCC-9.3 by typing

``` shell
$ cd hdf5-1.10.3 
$ ./configure --prefix=/cluster/home/huicheng/aster/public/hdf5-1.10.3
$ make -j 12
$make install
```
See the hdf5 libraries are created in /cluster/home/huicheng/aster/public/hdf5-1.10.3

    * 3.2

Go to med-4.0.0 folder to build he library in the same way by using hdf5 libraries

``` shell
cd ../med-4.0.0 $ ./configure --with-hdf5=/cluster/home/huicheng/aster/
public/hdf5-1.10.3 --prefix=/cluster/home/huicheng/aster
/public/med-4.0.0
make -j 12
make install
```

See the med libraries are created in /cluster/home/huicheng/aster/public/med-4.0.0 and return the main folder 

4.
Make the necessary changes in setup.cfg files to switch the Intel compiler.
PREFER_COMPILER = 'Intel'
Then type the python command to start main compilation
```
$python3 setup.py install --prefix=/cluster/home/huicheng/aster
```

When you have a warning about recompilation of hdf5/med  libraries, choose "no" and DO NOT recompile it In the end, all prerequisites should be assiged as "OK" and /cluster/projects/home/huicheng/aster/14.6 folder should be built with lib and bin folders I hope it works for you. 

Send me the setup.log file if it doesnt work.
Regards
Tufan
