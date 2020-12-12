# Singularity Notes

## smcpp

Create a Conda environment:

```
Bootstrap: library
From: centos:latest

%post
    yum -y update
    yum -y install wget
    wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
    chmod 755 Miniconda3-latest-Linux-x86_64.sh
    mkdir -p /software
    ./Miniconda3-latest-Linux-x86_64.sh -b -f -p /software/miniconda3
    source /software/miniconda3/etc/profile.d/conda.sh
    /software/miniconda3/bin/conda init bash
    /software/miniconda3/bin/conda create --name smcpp-env -c conda-forge -c terhorst smcpp -y
```

Create the image in the cloud:

```
$ singularity build --remote myimage.sif centos_smcpp.def
```

## gmsh

Needed to build image then run ldd on binary then figure out which package provided the missing libraries.

```
Bootstrap: library
From: ubuntu:18.04

%post
  apt-get -y update
  apt-get -y install wget libglu1-mesa libxrender1 libxcursor-dev
  apt-get -y install libxft2 lib32ncurses5 libxext6 libxinerama1

  wget https://gmsh.info/bin/Linux/gmsh-4.7.1-Linux64.tgz
  tar zxf gmsh-4.7.1-Linux64.tgz
  rm -rf gmsh-4.7.1-Linux64.tgz
```

```
[root] # singularity build gmsh.sif gmsh.def
[root] # chown jdh4 gmsh.sif; chgrp cses gmsh.sif
[jdh4] $ singularity shell gmsh.sif
[jdh4] $ /gmsh-4.7.1-Linux64/bin/gmsh -help
```

The above is a workaround for `libstdc++.so.6: version 'CXXABI_1.3.8' not found` and `libstdc++.so.6: version
'GLIBCXX_3.4.20' not found`. See 31735.


## dart

```
Bootstrap: docker
From: ubuntu:18.04

%post
  apt-get -y update && apt-get -y upgrade
  apt-get -y install build-essential cmake pkg-config git
  apt-get -y install libeigen3-dev libassimp-dev libccd-dev libfcl-dev libboost-regex-dev libboost-system-dev
  apt-get -y install libtinyxml2-dev liburdfdom-dev
  apt-get -y install libxi-dev libxmu-dev freeglut3-dev libopenscenegraph-dev
  apt-get -y install python3-pip
  apt-get -y update

  # Ubuntu 18.10 and older
  git clone https://github.com/pybind/pybind11 -b 'v2.2.4' --single-branch --depth 1
  cd pybind11
  mkdir build
  cd build
  cmake .. -DCMAKE_BUILD_TYPE=Release -DPYBIND11_TEST=OFF
  make -j4
  make install

  pip3 install dartpy

  git clone git://github.com/dartsim/dart.git
  cd dart
  git checkout tags/v6.8.2
  mkdir build
  cd build
  cmake ..
  make -j4
  make install
```

The above built successfully on Adroit:

```
[root@adroit4 tmp]# singularity build dart.img dart.recipe
```

## Serial Lammps

```
Bootstrap: library
From: ubuntu:18.04

%post
  apt-get -y update
  apt-get -y install build-essential cmake wget

  wget https://github.com/lammps/lammps/archive/stable_29Oct2020.tar.gz
  tar zxf stable_29Oct2020.tar.gz
  cd lammps-stable_29Oct2020
  mkdir build
  cd build
  cmake -D CMAKE_INSTALL_PREFIX=/opt \
  -D BUILD_MPI=no -D BUILD_OMP=no -D CMAKE_BUILD_TYPE=Release \
  -D CMAKE_CXX_FLAGS_RELEASE=-O3 -D PKG_MOLECULE=yes ../cmake
  make -j 10
  make install
```

To run the example:

```
$ singularity exec lammps.sif /opt/bin/lmp -in /lammps-stable_29Oct2020/examples/melt/in.melt
```
