# Building NAMD

## The basics

NAMD compilation is a two part process. You need a charm++ backend, on top of which you will compile the NAMD source itself. This means that there are two basic parts, broken up here into the charm++ component and the NAMD component. charm++ dictates the parallelism for NAMD, while the NAMD compilation is what links in accelerators.

## The simple example: single node x86_64 NAMD

We can do all of this at once in barely commented bash.
```bash
#wget a NAMD source tarball first. For licensing reasons, you need to get this yourself.
tar -zxf NAMD_2.14_Source.tar.gz
cd NAMD_2.14_Source
tar -xf charm-6.10.2.tar
cd charm-6.10.2
./build charm++ multicore-linux-x86_64 -j16 --with-production
```

The `build` script is very useful, and I encourage folks new to compiling NAMD to invoke it with `./build`, which takes you through a guided selection of different options for charm++. At the very end of the script, as it is asking you if you want to build now, note the single build line that charm++ recommends, which lets you skip this going forward, and is how I came up with the last line above.

```bash
cd .. #Go back to the NAMD source directory.
#NAMD depends on tcl and fftw. If you have library versions already installed, you *can* use them. For a minimum of fuss, you can pull the downloads directly from the NAMD developers.
wget http://www.ks.uiuc.edu/Research/namd/libraries/fftw-linux-x86_64.tar.gz
tar xzf fftw-linux-x86_64.tar.gz
mv linux-x86_64 fftw
wget http://www.ks.uiuc.edu/Research/namd/libraries/tcl8.5.9-linux-x86_64-threaded.tar.gz
tar xzf tcl8.5.9-linux-x86_64-threaded.tar.gz
mv tcl8.5.9-linux-x86_64-threaded tcl-threaded
#Start compiling NAMD
./config Linux-x86_64-g++.simple --charm-arch multicore-linux-x86_64
cd Linux-g++-x86_64.simple
make -j8
#A namd2 executable now should be in this directory.
```

## NAMD3 on Ubuntu 24.04 with CUDA

```bash
sudo apt 
tar -zxf NAMD_3.0_Source.tar.gz
cd NAMD_3.0_Source
tar -xf charm-8.0.0.tar
cd charm-8.0.0
./build charm++ multicore-linux-x86_64 --with-production -j8
cd ..
#Edit arch/Linux-x86_64.tcl to have TCLINCL=-I/usr/include/tcl8.6. You should also have fftw3 installed (libfftw3-dev)
./config Linux-x86_64-g++ --charm-arch multicore-linux-x86_64 --with-fftw3 --with-cuda --with-single-node-cuda
cd Linux-x86_64-g++/
make -j8
```

## NAMD on slurm-based clusters

You'll probably want a multinode executable. On my cluster, which uses infiniband and so UCX is the preferred backend, these are the steps.

```bash
#Get modules, or otherwise pick up CUDA and FFTW.
module load gompi/2023a CUDA/12.3.0 FFTW/3.3.10-GCC-12.3.0
#get your NAMD source again. This time from gitlab so we can also get NAMD3
git clone https://gitlab.com/tcbgUIUC/namd.git

cd namd
#Get the charm++ source
git clone https://github.com/UIUC-PPL/charm.git
cd charm
#Build charm++
./build charm++ ucx-linux-x86_64   smp  -j16  --with-production
cd ..
#Checkout the namd3.0 alpha (devel branch)
git checkout devel
#Config line is important! Without the with-single-node-cuda, you won't have CUDASOAIntegrate
./config Linux-x86_64-g++ --charm-arch ucx-linux-x86_64-smp --with-fftw3 --with-cuda --with-single-node-cuda
cd Linux-x86_64-g++
#Build NAMD
make -j8
#You should now have a namd3 executable.
```

## NAMD on GraceHopper

The big difference here is that the performance is trash unless you are very specific about your CUDA linking and gencode options.

```bash
./config Linux-ARM64-g++ --charm-arch multicore-linux-arm8 --with-single-node-cuda --with-cuda --with-fftw3 --cuda-gencode arch=compute_90,code=sm_90 --cuda-dlink arch=compute_90,code=sm_90
```

## NAMD on Frontier

```bash
#Multiple programming environments are available. First one I got to make sense was cray.
module load PrgEnv-cray/8.4.0
module load amd-mixed/5.7.0
module load cce/16.0.1
module load craype/2.7.23
module load cray-fftw
git clone git@gitlab.com:tcbgUIUC/namd.git
cd namd
git checkout devel
git clone https://github.com/UIUC-PPL/charm
cd charm
#CC and cc call the cray compilers.
env MPICXX=CC MPICC=cc ./buildold charm++ mpi-linux-x86_64 smp --with-production -j8
cd ../arch
cp Linux-x86_64-g++.arch Linux-x86_64-clang++.arch
#edit the created file to use craycxx and craycc instead of g++ and gcc.
#Edit the arch/Linux-x86_64.tcl file to point to the right place. By default it points to a non-existent file
cd ..
wget http://www.ks.uiuc.edu/Research/namd/libraries/tcl8.6.13-linux-x86_64-threaded.tar.gz
tar -zxf tcl8.6.13-linux-x86_64-threaded.tar.gz

#Remove -Xptxas from Make.depends, since the cray compilers don't have that option.
sed -i 's/-Xptxas//g` Make.depends
./config Linux-x86_64-clang++.mpi --charm-base ./charm --charm-arch mpi-linux-x86_64-smp --with-hip --rocm-prefix $ROCM_PATH --with-fftw3 --fftw-prefix $FFTW_ROOT --with-single-node-hip
cd Linux-x86_64-clang++.mpi
make -j8
```



## NAMD on Perlmutter

```bash
#Modules to use GNU for programming
module load PrgEnv-gnu
module load cray-fftw
module load cray-pmi
#get your NAMD source again. wget is fine
wget https://www.ks.uiuc.edu/Research/namd/3.0b6/...
tar -zxf NAMD_3.0_Source.tar.gz
cd NAMD_3.0_Source
tar -xf charm-8.0.0.tar
cd charm-8.0.0
./build charm++ ofi-crayshasta smp --with-production -j8
cd ..
wget http://www.ks.uiuc.edu/Research/namd/libraries/tcl8.6.13-linux-x86_64-threaded.tar.gz
tar -zxf tcl8.6.13-linux-x86_64-threaded.tar.gz

#For whatever reason, the build system isn't finding the NVHPCSDK directory as being set. So we set it.
#export NVHPCSDK_DIR=/opt/nvidia/hpc_sdk/Linux_x86_64/23.9
cd arch
#Edit the arch/Linux-x86_64.tcl file to point to the right place. By default it points to a non-existent file
#edit Linux-x86_64.cuda to ask for -lcufft_static_nocallback instead of -lcufft_static. This avoids a linking error.
cd ..
#Config line is important! Without the with-single-node-cuda, you won't have CUDASOAIntegrate
./config Linux-x86_64-g++ --charm-base ./charm-8.0.0 --charm-arch ofi-crayshasta-smp --with-cuda --with-fftw3 --fftw-prefix $FFTW_ROOT --with-single-node-cuda
#Build NAMD
cd Linux-x86_64-g++
make -j8
#You should now have a namd3 executable.
```
Note that Perlmutter has SMT turned on, so you should allocate as though there are 128 cores per GPU node (32 cores per GPU). This is an example submission script that is meant for replica exchange umbrella sampling:

```
#!/bin/bash
#SBATCH -A m3968_g
#SBATCH -C gpu
#SBATCH -J test
#SBATCH -o %x-%j.out
#SBATCH -t 0:10:00
#SBATCH -N 32
#SBATCH --gpus-per-node=4
#SBATCH --gpu-bind=single:1
#SBATCH -c 32
#SBATCH -n 128

module reset

#cd $MEMBERWORK/bip251/stmv
cd $SLURM_SUBMIT_DIR
module load PrgEnv-cray
module load cray-fftw
module load cray-pmi
NAMDDIR=/global/homes/j/jvermaas/NAMD_3.0b6_Source/Linux-x86_64-clang++.mpi
srun $NAMDDIR/namd3 +ppn 31 +ignoresharing +replicas 128 runtestSOA.namd +stdout runtestSOA-%d.log

```

## NAMD on Delta

This is really similar to Perlmutter and Frontier. They all currently use Slingshot 11, which means we need to use MPI to get reasonable performance. Note that Delta has SMT turned off, so there are only 64 cores per GPU node.
```bash
module load PrgEnv-gnu
module load craype-x86-milan
module load cray-fftw
module load cray-pmi
module load cuda/12.3.0

#get your NAMD source again.
#get your NAMD source again. wget is fine
wget https://www.ks.uiuc.edu/Research/namd/3.0b6/...
tar -zxf NAMD_3.0_Source.tar.gz
cd NAMD_3.0_Source
tar -xf charm-8.0.0.tar
cd charm-8.0.0
#The incdir is because the libfabric module doesn't set an include directory. This looks to be repeated alot...
./build charm++ ofi-linux-x86_64 cxi smp slurmpmi2cray --with-production -j8 --incdir=/opt/cray/libfabric/1.15.2.0/include --libdir=/opt/cray/libfabric/1.15.2.0/lib64 --incdir=/opt/cray/pe/pmi/6.1.13/include --libdir=/opt/cray/pe/pmi/6.1.13/lib
cd ..
wget http://www.ks.uiuc.edu/Research/namd/libraries/tcl8.6.13-linux-x86_64-threaded.tar.gz
tar -zxf tcl8.6.13-linux-x86_64-threaded.tar.gz
cd arch
#Edit the Linux-x86_64.tcl file to point to the right place. By default it points to a non-existent file
cd ..
./config Linux-x86_64-g++.cray --charm-base ./charm-8.0.0 --charm-arch ofi-linux-x86_64-smp-cxi-slurmpmi2cray --with-cuda --with-fftw3 --fftw-prefix $FFTW_ROOT --with-single-node-cuda
cd Linux-x86_64-g++.cray
#Build NAMD
make -j8
#Now you'd have a namd3 executable.
```

May need to do the following to resolve some goofy packages: `LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/cray/pe/lib64:/opt/cray/pe/fftw/3.3.10.5/x86_milan/lib`

## NAMD on Summit

On Summit, you likely want a multi-node executable, and everything is POWER9 instead of x86. There are also CUDA and fftw libraries to consider, which changes the build process a little.

```bash
CUDA_PREFIX=/sw/summit/cuda/10.1.168
module load cuda fftw
mkdir NAMDbuild
cd NAMDbuild
#get your NAMD source again. This time from github/gerrit so we can also get NAMD3
git clone https://github.com/UIUC-PPL/charm.git
cd charm
git checkout v6.10.2
./build charm++ pamilrts-linux-ppc64le   smp     -j16  --with-production
cd ..
git clone https://charm.cs.illinois.edu/gerrit/namd.git
cd namd
ln -s ../charm .
#Get NAMD 2.14
git checkout release-2-14
wget http://www.ks.uiuc.edu/Research/namd/libraries/tcl8.5.9-linux-ppc64le-threaded.tar.gz
tar xzf tcl8.5.9-linux-ppc64le-threaded.tar.gz
mv tcl8.5.9-linux-ppc64le-threaded tcl-threaded
./config Linux-POWER-g++.summit214 --charm-arch pamilrts-linux-ppc64le-smp --with-fftw3 --with-cuda --cuda-prefix $CUDA_PREFIX
cd Linux-POWER-g++.summit214
make -j8
#A namd2 executable will be in this directory now.
#Go back to the main directory for NAMD3
cd ..
```

For NAMD3, the build process is pretty similar, but the source code comes from another source. Beginning from the same terminal:

```bash
#Get the most recent alpha. "git checkout devel" is another option.
git checkout release-3-0-alpha-7
./config Linux-POWER-g++.summit30a7 --charm-arch pamilrts-linux-ppc64le-smp --with-fftw3 --with-cuda --cuda-prefix $CUDA_PREFIX --with-single-node-cuda
cd Linux-POWER-g++.summit30a7
make -j8
#The binary is now namd3
```

An example lsf script for Summit would look something like this:

```bash
#!/bin/bash
#BSUB -nnodes 2
#BSUB -W 0:10
#BSUB -P act000

module load cuda fftw
PATH=/gpfs/alpine/the/correct/path
NAMD_PATH=/home/path/to/namd
cd $PATH
jsrun -n12 -r6 -g1 -c7 --launch_distribution packed -b packed:7 $NAMD_PATH/namd3 +ignoresharing +ppn 6 +pemap 0-83:4,88-171:4 stmv.namd > stmv.log
```

The arguments for NAMD3 are a bit of an artform in and of themselves. Without the pemap, charm++ makes some unwise choices of what processors to bind. `+ppn 6` vs `+ppn 7` (or `+ppn 1` for namd3) is also a question worth testing. Adding specific communication threads with `+commap` arguments is also something to experiment with. A common example would be `+commap 0,28,56,88,116,144`.

