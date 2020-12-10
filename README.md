# Singularity Notes

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
