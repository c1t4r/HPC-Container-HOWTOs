# Bring Your Own Environment - Create Singularity Containers on your local computer

## About

This document shows by example how users can create their custom containers on their personal computers and export them to the HPC system JUSTUS.

## Prerequisites

You will need a personal computer with root access to install singularity and create containers. 
As of now the singularity runtime cannot be utilized fully under nonprivileged user accounts for that purpose, although this may become possible in future versions. This document uses version 2.3 of singularity.

### RPM package

[Download](../data/singularity-2.3-0.1.el7.x86_64.rpm?raw=true "singularity version 2.3 stable 64 bit") the rpm package.

This package is a custom build for RedHat 7 but should work also on CentOS, Fedora, OpenSuSE and similar distributions.
Install it using `yum install singularity-2.3-0.1.el7.x86_64.rpm`

### DEB package

[Download](../data/singularity-container_2.3-1_amd64.deb?raw=true "singularity version 2.3 stable 64 bit") the deb package.

This package is a custom build for Linux Mint 17 but should work also on Ubuntu 16.04 and Debian 7 and similar distributions.
Install it using `apt install singularity-container_2.3-1_amd64.deb`

## Creating a custom container

There are two main methods to create a custom container on a computer you have root access to.
1. Automatically - using a bootstrap file
2. Manually - step-by-step

We will demonstrate both methods using Ubuntu and the packages offered by its system repository to create a container running a MPI-enabled version of Gromacs.

### Manual creation of a Gromacs container

This method is usually the first attempt to create a custom container because it allows interactive installation, checks and improvements until it all works to satisfaction.
The procedure is as follows:
1. Create an empty container file
2. Choose a linux base system (CentOS, Ubuntu...)
3. Install and configure until it suits the purpose i.e. the desired software works
4. Add custom data/scripts to make it portable

At the beginning we create a container (1)

    cd /tmp
    singularity create -s 4096 testcontainer.img
    Creating 4096MiB image
    Binding image to loop
    Creating file system within image
    Image is done: testcontainer.img

Now we can import a standard Ubuntu into it (2)

    singularity import testcontainer.img docker://ubuntu:16.04
    Docker image path: index.docker.io/library/ubuntu:16.04
    Cache folder set to /home/ubuntu/.singularity/docker
    [5/5] |===================================| 100.0% 
    Importing: base Singularity environment
    Importing: /home/ubuntu/.singularity/docker/sha256:bd97b43c27e332fc4e00edf827bbc26369ad375187ce6eee91c616ad275884b1.tar.gz
    Importing: /home/ubuntu/.singularity/docker/sha256:6960dc1aba1816652969986284410927a5d942bf8042e077a3ebc8d1c58bb432.tar.gz
    Importing: /home/ubuntu/.singularity/docker/sha256:2b61829b0db5f4033ff48cbf3495271c8410c76e6396b56f15a79c3f7b5b7845.tar.gz
    Importing: /home/ubuntu/.singularity/docker/sha256:1f88dc826b144c661a8d1d08561e1ff3711f527042955505e9f3e563bdb2281f.tar.gz
    Importing: /home/ubuntu/.singularity/docker/sha256:73b3859b1e43f3ff32f10055951a568a9ad5ab6dc4ab61818b117b6912088f3d.tar.gz
    Importing: /home/ubuntu/.singularity/metadata/sha256:fe44851d529f465f9aa107b32351c8a0a722fc0619a2a7c22b058084fac068a4.tar.gz

Next, we start a root shell inside the container and install some software (3)

    sudo singularity exec -w testcontainer.img /bin/bash
    sudo: unable to resolve host meltingpot
    root@meltingpot:/tmp# id
    uid=0(root) gid=0(root) groups=0(root)

You should find you are root now inside the container (because of sudo)
    
    root@meltingpot:/tmp# apt update
    (...)
    root@meltingpot:/tmp# apt install gromacs gromacs-openmpi gromacs-data
    (...)
    
There is still plenty of empty space available to add software or user data/scripts to the container
    
    root@meltingpot:/tmp# df -h /
    Filesystem      Size  Used Avail Use% Mounted on
    /dev/loop0      3.9G  405M  3.3G  11% /
    root@meltingpot:/tmp# exit
    
Finally, download a sample gromacs file

    scp yourusername@justus:/opt/bwhpc/common/chem/gromacs/5.1.4/bwhpc-examples/ion_channel.tpr .
    
Enter the container using `bash`

    singularity exec testcontainer.img /bin/bash

You should find that you are now standard user inside your container and that the contents of `/tmp` are shared between host and container

    stefan@meltingpot:/tmp$ id
    uid=1000(stefan) gid=1000(stefan) groups=1000(stefan),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),107,110,125,500090(vasp_users)
    stefan@meltingpot:/tmp$ ls 
    firefox_stefan	hsperfdata_root  mintUpdate			       orbit-stefan  singularity-container_2.2-1_amd64.deb  singularity_2.2-1_amd64.changes  ssh-NUEP9lMyKHDt	tmp.uH2YJ6jpR1	tmpd_fYpy
    gpg-2xGcVB	ion_channel.tpr  openmpi-sessions-stefan@meltingpot_0  singularity   singularity-test.iDusMr		    sni-qt_keepassx_8967-rlxQtE      testcontainer.img	tmpGHNsdK

Now source the gromacs environment and run a short test

    stefan@meltingpot:/tmp$ source /usr/share/gromacs/shell-specific/GMXRC.bash
    stefan@meltingpot:/tmp$ OMP_NUM_THREADS=1 mpirun -n 2 mdrun_mpi.openmpi mdrun -s ion_channel.tpr -maxh 0.50 -noconfout -nsteps 500 -g logfile -v > mdrun.out
    (...)
    
While the test is running open up another terminal on your machine and verify using `top` that the MPI processes are actually running

    25328 stefan    39  19  534760 145088  15576 R 100,0  1,8   1:35.90 mdrun_mpi_d.ope                                                                                                                                                          
    25329 stefan    39  19  527960 139272  15252 R 100,0  1,8   1:35.90 mdrun_mpi_d.ope

After a few minutes the test should have finished and you can exit the container. Note that performance may be worse than what you expected due to lacking AVX2 support!

    stefan@meltingpot:/tmp$ exit

Now we are preparing the container for export to JUSTUS (4)

A few things must still be done to be able to redistribute this container:
1. Include example run script
2. Include example data file

Singularity containers have an default run script under `/singularity`
This is invoked if we execute the container like so

    chmod +x testcontainer.img
    ./testcontainer.img
    
So let's create a run script

    cat <<EOF > singularity
    #!/bin/bash
    source /usr/share/gromacs/shell-specific/GMXRC.bash
    OMP_NUM_THREADS=1 mpirun -n 2 mdrun_mpi_d.openmpi mdrun -s /data/ion_channel.tpr -maxh 0.50 -noconfout -nsteps 500 -g logfile -v > /tmp/mdrun.out
    EOF
    chmod 755 ./singularity 

and copy it into the container

    sudo singularity copy testcontainer.img ./singularity /singularity
    
Copy the data file

    sudo singularity exec -w testcontainer.img mkdir /data
    sudo singularity copy testcontainer.img ion_channel.tpr /data/ion_channel.tpr
    
Test-run the container (using two MPI threads)

    time ./testcontainer.img
    (...)
    real	2m43.174s
    user	5m25.476s
    sys	0m0.252s
    
Now copy this container to justus and run it there as well

    rsync -rahzvP testcontainer.img justus:
    
    ssh yourusername@justus
    ssh n0726
    time ./testcontainer.img
    (...)
    real	2m21.810s
    user	4m43.179s
    sys	0m0.436s

### Automatic creation of a Gromacs container using a bootstrap file

Bootstrap files serve as singularity container building scripts (comparable to what Dockerfiles are for Docker). 
Once the installation procedure for a certain container has somewhat matured it is useful to create a bootstrap 
file because it dramatically simplifies the rebuilding process for a custom container especially for end users.

This is a bootstrap definition file for the previous gromacs container:

    cat << EOF > gromacs_ubuntu.def
    BootStrap: docker
    From: ubuntu:16.04
    IncludeCmd: yes

    %runscript
    #!/bin/bash
    source /usr/share/gromacs/shell-specific/GMXRC.bash
    OMP_NUM_THREADS=1 mpirun -n 2 mdrun_mpi_d.openmpi \
    mdrun -s /data/ion_channel.tpr -maxh 0.50 -noconfout -nsteps 500 -g logfile -v > /tmp/mdrun.out

    %setup
    mkdir -p $SINGULARITY_ROOTFS/data
    cp ./ion_channel.tpr $SINGULARITY_ROOTFS/data
    
    %files
    ./ion_channel.tpr /data/ion_channel.tpr

    %post
    apt -y update
    apt -y install gromacs gromacs-openmpi gromacs-data
    mkdir -p /data
    EOF

Copy and paste this snippet in your shell to create `gromacs_ubuntu.def` and simply run

    singularity create testcontainer2.img
    sudo singularity bootstrap testcontainer2.img gromacs_ubuntu.def
    
to create the same container in a single step.
[Here](http://singularity.lbl.gov/bootstrap-image) you can find more info about the bootstrap file format definition and how to create your own bootstrap definitions.
This should complete without errors, and then you can execute the container as the previous one:

    time ./testcontainer2.img
    (...)
    real	2m24.789s
    user	4m49.095s
    sys	0m0.479s

    
    
Congratulations, you are finished :)
