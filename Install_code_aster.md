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
sudo apt-get install gcc g++ gfortran cmake python3 python3-dev python3-numpy tk bison flex liblapack-dev libblas-dev libboost-python-dev libboost-numpy-dev zlib1g-dev xterm nedit ddd xemacs21 kwrite gedit gnome-terminal
```
### 2.1 Install python module
We suggest install pip and pip3 first.
```shell
   pip install numpy setuptools matplotlib
```
**NOTE:** If it raise some problem about "blas", please make sure the ```libopenblas-base libopenblas-dev``` is not instlled with ```libblas-dev``` at the same time.

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

**NOTE:** you can test your installnation with the following command:
```shell
/opt/aster/bin/as_run --test sdnl142a
```

