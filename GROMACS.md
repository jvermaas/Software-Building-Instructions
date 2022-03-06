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
cmake .. -DGMX_BUILD_OWN_FFTW=ON -DGMX_GPU=CUDA -DCMAKE_INSTALL_PREFIX=/global/homes/j/jvermaas/modules/gromacs-2021.4 -DCMAKE_C_COMPILER=gcc -DCMAKE_CXX_COMPILER=g++
make -j8
make -j8 install
```
Example on my local cluster
```bash
module load CUDA/11.4.2
module load gompi/2021b
module load FFTW
module load Python/3.9.6
module load cmake
wget https://ftp.gromacs.org/gromacs/gromacs-2022.tar.gz
tar -zxf gromacs-2022.tar.gz 
cd gromacs-2022/
mkdir build
cd build
cmake .. -DGMX_GPU=CUDA -DGMX_FFT_LIBRARY=fftw3 -DCMAKE_INSTALL_PREFIX=/mnt/home/vermaasj/gromacs/2022
make -j 8
make install

cd ..
mkdir buildmpi
cmake .. -DGMX_GPU=CUDA -DGMX_FFT_LIBRARY=fftw3 -DCMAKE_INSTALL_PREFIX=/mnt/home/vermaasj/gromacs/2022 -DGMX_MPI=ON
make -j 8
make install
```
