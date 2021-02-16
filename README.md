
## MKL for .deb-based systems: An easy recipe

This post describes how to _easily_ install the 
[Intel Math Kernel Library (MKL)](https://software.intel.com/en-us/mkl?cid=sem43700010399172562&intel_term=%2Bintel%20%2Bmkl&gclid=Cj0KCQjwzcbWBRDmARIsAM6uChXqzD4ACUJqCiu3zRJKA9rkC31XOhm9lIkEYiwBITMR_8hJbIAExF8aAn_LEALw_wcB&gclsrc=aw.ds) on a Debian or Ubuntu system. 
Very good basic documentation is provided by Intel [at their
site](https://software.intel.com/en-us/articles/installing-intel-free-libs-and-python-apt-repo). The discussion here
is more narrow as it focusses just on the Math Kernel Library (MKL).

The `tl;dr` version:  Use [this script](script.sh) which contains the commands described here.  

### First Step: Set up apt

We download the GnuPG key first and add it to the keyring:

```sh
cd /tmp
wget https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS-2019.PUB
apt-key add GPG-PUB-KEY-INTEL-SW-PRODUCTS-2019.PUB
```

To add all Intel products we would run first command, but here we focus just on the MKL.  The website above
lists other suboptions (TBB, DAAL, MPI, ...)

```sh
## all products:
#wget https://apt.repos.intel.com/setup/intelproducts.list -O /etc/apt/sources.list.d/intelproducts.list

## just MKL
sh -c 'echo deb https://apt.repos.intel.com/mkl all main > /etc/apt/sources.list.d/intel-mkl.list'
```

We then update our lists of what is available in the repositories.

```sh
apt-get update
```

### Install MKL

Now that we have everything set up, installing the MKL is as simple as:

```sh
apt-get install intel-mkl-64bit-2020.2-108
```

This picks the 64-bit only variant of the (currently) most recent builds.      

### Integrate MKL

One of the key advantages of a Debian or Ubuntu system is the overall integration providing a raft of
useful features.  One of these is the seamless and automatic selection of alternatives.  By
declaring a particular set of BLAS and LAPACK libraries the default, _all_ application linked
against this interface will use the default.  Better still, users can switch between these as well.

So here we can make the MKL default for BLAS and LAPACK:

```sh
## update alternatives
update-alternatives --install /usr/lib/x86_64-linux-gnu/libblas.so     libblas.so-x86_64-linux-gnu      /opt/intel/mkl/lib/intel64/libmkl_rt.so 150
update-alternatives --install /usr/lib/x86_64-linux-gnu/libblas.so.3   libblas.so.3-x86_64-linux-gnu    /opt/intel/mkl/lib/intel64/libmkl_rt.so 150
update-alternatives --install /usr/lib/x86_64-linux-gnu/liblapack.so   liblapack.so-x86_64-linux-gnu    /opt/intel/mkl/lib/intel64/libmkl_rt.so 150
update-alternatives --install /usr/lib/x86_64-linux-gnu/liblapack.so.3 liblapack.so.3-x86_64-linux-gnu  /opt/intel/mkl/lib/intel64/libmkl_rt.so 150
update-alternatives --install /usr/lib/x86_64-linux-gnu/libopenblas.so libopenblas.so-x86_64-linux-gnu  /opt/intel/mkl/lib/intel64/libmkl_rt.so 150
update-alternatives --install /usr/lib/x86_64-linux-gnu/libopenblas.so.0 libopenblas.so.0-x86_64-linux-gnu  /opt/intel/mkl/lib/intel64/libmkl_rt.so 150
```

Next, we have to tell the dyanmic linker about two directories use by the MKL, and have it update
its cache:

```sh
echo "/opt/intel/lib/intel64"     >  /etc/ld.so.conf.d/mkl.conf
echo "/opt/intel/mkl/lib/intel64" >> /etc/ld.so.conf.d/mkl.conf
ldconfig
```

### Set an Environment Variable

As discussed in [issue ticket #2](https://github.com/eddelbuettel/mkl4deb/issues/2),
mixing Intel OpenMP and GNU OpenMP run-times in one application can lead to issues. 

Since most of the open-source performance libraries use GNU OpenMP it is safer to make the MKL
library also use GNU OpenMP as well by setting `MKL_THREADING_LAYER=GNU` in either
`/etc/environment` or your local per-user settings. 

Here we use the file in `/etc`:

```sh
echo "MKL_INTERFACE_LAYER=GNU,LP64" >> /etc/environment
echo "MKL_THREADING_LAYER=GNU" >> /etc/environment
```

Thanks to Evarist Fomenko from Intel's Novosibirsk office for help with this point.


### Use the MKL

Now the MKL is 'known' and the default. If we start R, its `sessionInfo()` shows the MKL:

```sh
Matrix products: default                            
BLAS/LAPACK: /opt/intel/compilers_and_libraries_2018.2.199/linux/mkl/lib/intel64_lin/libmkl_rt.so
```

### Benchmarks


```r
# Vanilla r-base Rocker with default reference BLAS 
> n <- 1e3 ; X <- matrix(rnorm(n*n),n,n);  system.time(svd(X)) 
   user  system elapsed 
  2.239   0.004   2.266 
> 

# OpenBlas added to r-base Rocker
>  n <- 1e3 ; X <- matrix(rnorm(n*n),n,n);  system.time(svd(X)) 
   user  system elapsed 
  1.367   2.297   0.353 
> 

# MKL added to r-base Rocker
> n <- 1e3 ; X <- matrix(rnorm(n*n),n,n)  
> system.time(svd(X))                               
   user  system elapsed                             
  1.772   0.056   0.350                             
>  
```

So just R (with reference BLAS) is slow. (Using Docker is done here to have
clean comparisons while not altering the outer host system; impact of running
Docker on Linux should be minimal.) Adding OpenBLAS helps quite a bit already
by offering multi-core processing -- the, and MKL does not yet improve
materially over OpenBLAS.  Now, this of course was not any serious
benchmarking---we just ran one SVD. More to do as time permits...

### Removal, if needed

Another rather nice benefit of the package management is that clean
removal is also possible:

```sh
root@c9f8062fbd93:/tmp# apt-get autoremove intel-mkl-64bit-2018.2-046
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following packages will be REMOVED:
  intel-comp-l-all-vars-18.0.2-199 intel-comp-nomcu-vars-18.0.2-199 intel-mkl-64bit-2018.2-046 
  intel-mkl-cluster-2018.2-199 intel-mkl-cluster-c-2018.2-199 intel-mkl-cluster-common-2018.2-199 
  intel-mkl-cluster-f-2018.2-199 intel-mkl-cluster-rt-2018.2-199 intel-mkl-common-2018.2-199 
  intel-mkl-common-c-2018.2-199 intel-mkl-common-c-ps-2018.2-199 intel-mkl-common-f-2018.2-199 
  intel-mkl-common-ps-2018.2-199 intel-mkl-core-2018.2-199 intel-mkl-core-c-2018.2-199 
  intel-mkl-core-f-2018.2-199 intel-mkl-core-ps-2018.2-199 intel-mkl-core-rt-2018.2-199 
  intel-mkl-doc-2018 intel-mkl-doc-ps-2018 intel-mkl-f95-2018.2-199 intel-mkl-f95-common-2018.2-199 
  intel-mkl-gnu-2018.2-199 intel-mkl-gnu-c-2018.2-199 intel-mkl-gnu-f-2018.2-199 intel-mkl-gnu-f-rt-2018.2-199 
  intel-mkl-gnu-rt-2018.2-199 intel-mkl-pgi-2018.2-199 intel-mkl-pgi-c-2018.2-199 intel-mkl-pgi-f-2018.2-199 
  intel-mkl-pgi-rt-2018.2-199 intel-mkl-psxe-2018.2-046 intel-mkl-tbb-2018.2-199 intel-mkl-tbb-rt-2018.2-199 
  intel-openmp-18.0.2-199 intel-psxe-common-2018.2-046 intel-psxe-common-doc-2018 intel-tbb-libs-2018.2-199 
  intel-tbb-libs-32bit-2018.2-199 libisl15
0 upgraded, 0 newly installed, 40 to remove and 0 not upgraded.
After this operation, 1,904 kB disk space will be freed.
Do you want to continue? [Y/n] n                    
Abort.                                              
root@c9f8062fbd93:/tmp#  
```

where we said 'no' just to illustrate the option.

As a second step, you want to also update the _alternatives_ setting via 

```sh
update-alternatives --remove libblas.so-x86_64-linux-gnu     \
                             /opt/intel/mkl/lib/intel64/libmkl_rt.so 
update-alternatives --remove libblas.so.3-x86_64-linux-gnu   \
                             /opt/intel/mkl/lib/intel64/libmkl_rt.so 
update-alternatives --remove liblapack.so-x86_64-linux-gnu   \
                             /opt/intel/mkl/lib/intel64/libmkl_rt.so 
update-alternatives --remove liblapack.so.3-x86_64-linux-gnu \
                             /opt/intel/mkl/lib/intel64/libmkl_rt.so 
```


### Summary

Package management systems are fabulous.  Kudos to Intel for supporting `apt` (and also `yum` in case you are on an rpm-based system).
We can install the MKL with just a few commands (which we regrouped in [this script](script.sh)).   

The MKL has a serious footprint with an installed size of just under 2gb.  But for those doing extended amounts of numerical analysis, 
installing this library may well be worth it.
