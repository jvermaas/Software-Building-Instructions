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

## NAMD on slurm-based clusters

You'll probably want a multinode executable. On my cluster, which uses infiniband and so UCX is the preferred backend, these are the steps.

```bash
#Get modules, or otherwise pick up CUDA and FFTW.
module load gompi/2020a CUDA FFTW3
#get your NAMD source again. This time from gitlab so we can also get NAMD3
git clone https://gitlab.com/tcbgUIUC/namd.git

cd namd
#Get the charm++ source
git clone https://github.com/UIUC-PPL/charm.git
cd charm
git checkout v6.10.2
#Build charm++
./build charm++ ucx-linux-x86_64   smp  -j16  --with-production
cd ..
#Checkout the namd3.0 alpha (devel branch)
git checkout release-3-0-alpha-9
#Config line is important! Without the with-single-node-cuda, you won't have CUDASOAIntegrate
./config Linux-x86_64-g++ --charm-arch ucx-linux-x86_64-smp --with-fftw3 --with-cuda --with-single-node-cuda
cd Linux-x86_64-g++
#Build NAMD
make -j8
#You should now have a namd3 executable.
```

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
