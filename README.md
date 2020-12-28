# Singularity Notes

## Examples

[https://github.com/mkandes/naked-singularity/tree/master/definition-files](https://github.com/mkandes/naked-singularity/tree/master/definition-files)  
[https://www.rocker-project.org](https://www.rocker-project.org)  
[https://github.com/sylabs/examples](https://github.com/sylabs/examples)  

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

```
Bootstrap: docker
From: ubuntu:latest

%environment
    export PATH="/opt/miniconda3/bin:${PATH}"

%post
    apt-get -y update && apt-get -y upgrade
    apt-get -y install wget
    #wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
    wget https://repo.anaconda.com/miniconda/Miniconda3-py37_4.9.2-Linux-x86_64.sh
    chmod 755 Miniconda3-py37_4.9.2-Linux-x86_64.sh
    mkdir -p /opt/miniconda3
    ./Miniconda3-py37_4.9.2-Linux-x86_64.sh -b -f -p /opt/miniconda3
    /opt/miniconda3/bin/conda install -c conda-forge -c terhorst -y smcpp
```

Create the image in the cloud:

```
$ singularity build --remote myimage.sif centos_smcpp.def
```

## TensorFlow CPU on notexa

```
Bootstrap: docker
From: ubuntu:latest

%post
    apt-get -y update
    apt-get -y install wget
    wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
    chmod 755 Miniconda3-latest-Linux-x86_64.sh
    mkdir -p /software
    ./Miniconda3-latest-Linux-x86_64.sh -b -f -p /software/miniconda3
    . /software/miniconda3/etc/profile.d/conda.sh
    /software/miniconda3/bin/conda init bash
    /software/miniconda3/bin/conda create --name tf2-cpu tensorflow -y
```

```
$ singularity exec tf2-cpu.sif /software/miniconda3/envs/tf2-cpu/bin/python3 -c "import tensorflow as tf; tf.keras.datasets.mnist.load_data()"
$ git clone https://github.com/PrincetonUniversity/slurm_mnist
$ singularity exec tf2-cpu.sif /software/miniconda3/envs/tf2-cpu/bin/python3 ./slurm_mnist/mnist_classify.py
Test accuracy: 0.9732999801635742
```

## R

```
Bootstrap: docker
From: ubuntu:latest

%files
    r.def /opt
    main.R

%post
    apt-get -y update
    DEBIAN_FRONTEND=noninteractive apt-get -y install build-essential r-base
    apt-get -y install locales
    apt-get clean
    locale-gen en_US.UTF-8
    R --slave -e 'install.packages(c("dplyr", "caret"))'

%test
    #!/bin/bash
    exec R --slave -e "library(caret); installed.packages();"

%runscript
    #!/bin/bash
    Rscript --slave "main.R"
```

The above produced a container with R 3.6. Looks like need to add a repo to get 4.0. The container was made on notexa and it worked on mbp2019.

## RStudio

```
Bootstrap: docker
From: ubuntu:latest

%post
    apt-get -y update
    DEBIAN_FRONTEND=noninteractive apt-get -y install build-essential wget vim nano git python python-dev bzip2 r-base
    apt-get -y install locales libcurl4-openssl-dev libv8-dev libgeos-dev libgdal-dev libproj-dev
    apt-get -y install protobuf-compiler libudunits2-dev libprotobuf-dev libjq-dev libfontconfig1-dev libcairo2-dev
    apt-get -y install xvfb xauth xorg-dev libx11-dev libglu1-mesa-dev xfonts-base

    #wget https://download2.rstudio.org/server/bionic/amd64/rstudio-server-1.3.1093-amd64.deb
    #gdebi -n rstudio-server-1.3.1093-amd64.deb

    apt-get -y install gdebi-core
    wget https://download1.rstudio.org/desktop/bionic/amd64/rstudio-1.3.1093-amd64.deb
    gdebi -n rstudio-1.3.1093-amd64.deb

    apt-get clean
    locale-gen en_US.UTF-8
```

```
$ singularity shell myimage.sif 
Singularity> rstudio
QStandardPaths: XDG_RUNTIME_DIR points to non-existing path '/run/user/150340', please create it with 0700 permissions.
qt.qpa.xcb: QXcbConnection: XCB error: 3 (BadWindow), sequence: 1347, resource id: 12916261, major code: 40 (TranslateCoords), minor code: 0
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
