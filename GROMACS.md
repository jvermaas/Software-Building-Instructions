# GROMACS on Perlmutter (and other slurm-based clusters)

We'd need to download a recent GROMACS package, and then compile it.
On Perlmutter
```bash
wget https://ftp.gromacs.org/gromacs/gromacs-2021.4.tar.gz
tar -zxf gromacs-2021.4.tar.gz
module load cmake/3.20.5
module load gcc/10.3.0
module load cuda/11.2.2
cd gromacs-2021.4
mkdir build
cd build
cmake .. -DGMX_BUILD_OWN_FFTW=ON -DGMX_GPU=CUDA -DCMAKE_PREFIX_PATH=/global/homes/j/jvermaas/modules/gromacs-2021.4 -DCMAKE_C_COMPILER=gcc -DCMAKE_CXX_COMPILER=g++
make -j8
make install
```
