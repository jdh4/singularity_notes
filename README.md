# Singularity Notes

## Examples

[https://github.com/mkandes/naked-singularity/tree/master/definition-files](https://github.com/mkandes/naked-singularity/tree/master/definition-files)  
[https://www.rocker-project.org](https://www.rocker-project.org)  
[https://github.com/sylabs/examples](https://github.com/sylabs/examples)  
[https://github.com/NIH-HPC/Singularity-Tutorial](https://github.com/NIH-HPC/Singularity-Tutorial)  
QUAY.io, BioContainers

## alias

```
Bootstrap: docker
From: ubuntu:20.04

%post
  echo '#!/bin/bash' > myaliases.sh
  echo 'alias ll="ls -l"' >> myaliases.sh
```

```
$ singularity shell myalias.sif
Singularity> . /myaliases.sh 
Singularity> ll
```

## GPU kernel

```
$ singularity pull docker://nvidia/cuda:11.1.1-devel-ubuntu20.04
$ singularity shell --nv ~/software/cuda_11.1.1-devel-ubuntu20.04.sif
Singularity> nvcc hello_world_gpu.cu
Singularity> ./a.out 
Hello world from the CPU.
Hello world from the GPU.
```

## anaconda3

How to source .bashrc without it finding host version?

```
Bootstrap: docker
From: ubuntu:20.04

%environment
  PATH=$PATH:/opt/anaconda3/bin
  export PATH

%post
  apt-get -y update && apt-get -y upgrade
  apt-get -y install git bzip2 wget
 
  wget https://repo.anaconda.com/archive/Anaconda3-2020.11-Linux-x86_64.sh
  bash Anaconda3-2020.11-Linux-x86_64.sh -b -p /opt/anaconda3
  echo "PATH=/opt/anaconda3/bin:\$PATH" >> $HOME/.bashrc
  echo "export PATH" >> $HOME/.bashrc
  rm Anaconda3-2020.11-Linux-x86_64.sh
  /opt/anaconda3/bin/conda install --channel gurobi gurobi -y
  
  # cleanup
  apt-get -y autoremove --purge
  apt-get -y clean
```

## HW

```
Bootstrap: docker
From: ubuntu:18.04

%environment
    export MY_VAR='---This is my var.---'

%post
    apt-get -qq -y update
    apt-get -qq -y install python > /dev/null

%files
    env_print.py    /

%runscript
    python /env_print.py
```

## afni

```
Bootstrap: docker
From: ubuntu:20.04

%environment
  # Set system locale
  export LC_ALL='C'
  export PATH=$PATH:/root/abin

%post -c /bin/bash
  export DEBIAN_FRONTEND=noninteractive
  apt-get -y update && apt-get -y upgrade
  apt-get install -y software-properties-common
  add-apt-repository universe

  apt-get install -y tcsh xfonts-base libssl-dev  \
		python-is-python3                 \
		python3-matplotlib                \
		libgsl-dev \
                netpbm gnome-tweak-tool   \
		libjpeg62 xvfb xterm vim curl     \
		gedit evince eog                  \
		libglu1-mesa-dev libglw1-mesa     \
		libxm4 build-essential            \
		libcurl4-openssl-dev libxml2-dev  \
		libgfortran-8-dev libgomp1        \
		gnome-terminal nautilus           \
		gnome-icon-theme-symbolic         \
		firefox xfonts-100dpi             \
                libv8-dev libudunits2-dev \
		r-base-dev

  ln -s /usr/lib/x86_64-linux-gnu/libgsl.so.23 /usr/lib/x86_64-linux-gnu/libgsl.so.19

  echo $0

  cd $HOME
  curl -O https://afni.nimh.nih.gov/pub/dist/bin/misc/@update.afni.binaries
  tcsh @update.afni.binaries -package linux_ubuntu_16_64 -do_extras

  cp $HOME/abin/AFNI.afnirc $HOME/.afnirc
  export PATH=$PATH:/root/abin

  echo $0

  chmod -R 775 /root
  suma -update_env

  echo 'setenv PATH $PATH\:/root/abin' >> /root/.cshrc
  echo 'export PATH=$PATH:/root/abin'  >> /root/.bashrc
  cat /root/.bashrc

  export R_LIBS=$HOME/R
  mkdir  $R_LIBS
  echo  'setenv R_LIBS ~/R'     >> ~/.cshrc
  echo  'export R_LIBS=$HOME/R' >> ~/.bashrc

  #rPkgsInstall -pkgs ALL

  #python afni_system_check.py -check_all

  echo 'set filec'    >> ~/.cshrc
  echo 'set autolist' >> ~/.cshrc
  echo 'set nobeep'   >> ~/.cshrc

  echo 'alias ls ls --color=auto' >> ~/.cshrc
  echo 'alias ll ls --color -l'   >> ~/.cshrc
  echo 'alias ltr ls --color -ltr'   >> ~/.cshrc
  echo 'alias ls="ls --color"'    >> ~/.bashrc
  echo 'alias ll="ls --color -l"' >> ~/.bashrc
  echo 'alias ltr="ls --color -ltr"' >> ~/.bashrc
```

## sft

```
Bootstrap: docker
From: ubuntu:20.04

%environment
    # Set system locale
    export LC_ALL='C'

%post
    export DEBIAN_FRONTEND=noninteractive
    apt-get -y update && apt-get -y upgrade
    apt-get install -y curl
    apt-get install -y software-properties-common
    apt-get install -y openssh-server
    
    echo "deb http://pkg.scaleft.com/deb linux main" | tee -a /etc/apt/sources.list
    curl -C - https://dist.scaleft.com/pki/scaleft_deb_key.asc | apt-key add -
    apt-get -y update
    apt-get install -y scaleft-client-tools
```

```
$ singularity remote login SylabsCloud
$ singularity build --remote sft.sif recipe.def
```

## eog

```
Bootstrap: docker
From: ubuntu:latest

%environment
    # Set system locale
    export LC_ALL='C'
    export XDG_RUNTIME_DIR=/tmp

%post
    export DEBIAN_FRONTEND=noninteractive
    apt-get -y update && apt-get -y upgrade
    apt-get -y install eog xorg gnuplot binutils
    strip --remove-section=.note.ABI-tag /usr/lib/x86_64-linux-gnu/libQt5Core.so.5
    mkdir -p /run/user/1000/dconf
    chmod -R 777 /run
```

In the above, I don't have `XDG_RUNTIME_DIR` and `/run/user/1000/` worked out perfectly. Could try using `--bind /tmp:/run/user/1000`

If do nothing with `XDG_RUNTIME_DIR` and `/run/user/1000/` then get this error when launching eog:

```
(eog:23952): dconf-CRITICAL **: 10:29:33.980: unable to create directory '/run/user/1000/dconf': Read-only file system.  dconf will not work properly.
```

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
From: r-base:4.0.2

%post
  R --slave -e 'install.packages(c("caret"))'

%runscript
  #!/bin/bash
  Rscript --slave -e "library(caret); installed.packages();"
```

You can install packages locally and then specify where they are when used later:

```
> library(libridate)
Error in library(libridate) : there is no package called ‘libridate’
> library("lubridate", lib.loc="~/R/x86_64-pc-linux-gnu-library/4.0")
```

```
$ singularity exec myimage.sif cat /etc/os-release
PRETTY_NAME="Debian GNU/Linux bullseye/sid"
NAME="Debian GNU/Linux"
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
```

[https://hub.docker.com/r/rocker/r-ubuntu](https://hub.docker.com/r/rocker/r-ubuntu)

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
Bootstrap: docker
From: ubuntu:20.10

%post
  export DEBIAN_FRONTEND=noninteractive
  apt-get -y update && apt-get -y upgrade
  apt-get -y install libfreetype-dev
  apt-get -y install libocct-foundation-dev libocct-data-exchange-dev
  #apt-get -y install libfltk1.3-dev
  apt-get -y install libjpeg-dev
  apt-get -y install xorg
  apt-get -y install libhdf5-dev
  apt-get -y install libgmp-dev
  apt-get -y install build-essential cmake
  apt-get -y install git wget

  wget https://www.fltk.org/pub/fltk/1.3.4/fltk-1.3.4-2-source.tar.gz
  tar zxf fltk-1.3.4-2-source.tar.gz
  cd fltk-1.3.4-2
  ./configure
  make -j 4
  make install
 
  #apt-get -y install wget libglu1-mesa libxrender1 libxcursor-dev
  #apt-get -y install libxft2 lib32ncurses5 libxext6 libxinerama1

  git clone http://gitlab.onelab.info/gmsh/gmsh.git
  cd gmsh
  mkdir build && cd build
  cmake .. -DCMAKE_INSTALL_PREFIX=/opt
  make -j 4
  make install
```

To launch the GUI:

```
$ singularity exec gmsh /opt/bin/gmsh
```


Old stuff below:

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
