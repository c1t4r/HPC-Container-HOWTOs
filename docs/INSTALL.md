# Installation and Creation of an example container

## Install singularity on your personal computer

### RPM package

[Download](../data/singularity-2.2.1-0.1.el7.x86_64.rpm "singularity version 2.2.1 stable 64 bit") the rpm package

This package is a custom build for RedHat 7 but should work also on CentOS, Fedora, OpenSuSE and similar distributions.

### DEB package

[Download](../data/singularity-container_2.2-1_amd64.deb "singularity version 2.2.1 stable 64 bit") the deb package

This package is a custom build for Linux Mint 17 but should work also on Ubuntu 16.04 and Debian 7 and similar distributions.

## Create a gromacs container

At the beginning we create a container

    cd /tmp
    sudo singularity create -s 4096 testcontainer.img
    Creating a new image with a maximum size of 4096MiB...
    Executing image create helper
    Formatting image with ext3 file system
    Done.`

Now we can import a standard Ubuntu into it

    sudo singularity import testcontainer.img docker://ubuntu:16.04
    Downloading layer: sha256:a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4
    Downloading layer: sha256:cf9722e506aada1109f5c00a9ba542a81c9e109606c01c81f5991b1f93de7b66
    Downloading layer: sha256:3deef3fcbd3072b45771bd0d192d4e5ff2b7310b99ea92bce062e01097953505
    Downloading layer: sha256:2955fb827c947b782af190a759805d229cfebc75978dba2d01b4a59e6a333845
    Downloading layer: sha256:55010f332b047687e081a9639fac04918552c144bc2da4edb3422ce8efcc1fb1
    Downloading layer: sha256:b6f892c0043b37bd1834a4a1b7d68fe6421c6acbc7e7e63a4527e1d379f92c1b
    Adding Docker CMD as Singularity runscript...
    Bootstrap initialization
    No bootstrap definition passed, updating container
    Executing Prebootstrap module
    Executing Postbootstrap module
    Done.

Next, we start a root shell inside the container and install some software

    sudo singularity exec -w testcontainer.img /bin/bash
    root@meltingpot:/tmp# id
    uid=0(root) gid=0(root) groups=0(root)

You should find you are root now inside the container (because of sudo)
    
    root@meltingpot:/tmp# apt update
    (...)
    root@meltingpot:/tmp# apt install gromacs gromacs-openmpi gromacs-data
    (...)
    root@meltingpot:/tmp# exit
    
Finally, download a sample gromacs file

    scp yourusername@justus:/opt/bwhpc/common/chem/gromacs/5.1.4/bwhpc-examples/ion_channel.tpr .
    singularity exec testcontainer.img /bin/bash
    groups: cannot find name for group ID 107
    groups: cannot find name for group ID 110
    groups: cannot find name for group ID 125
    stefan@meltingpot:~$ id
    uid=1000(stefan) gid=1000(stefan) groups=1000(stefan),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),107,110,125,500090(vasp_users)

You should find that you are now standard user inside your container and that the contents of `/tmp` are shared between host and container

    stefan@meltingpot:/tmp$ ls 
    firefox_stefan	hsperfdata_root  mintUpdate			       orbit-stefan  singularity-container_2.2-1_amd64.deb  singularity_2.2-1_amd64.changes  ssh-NUEP9lMyKHDt	tmp.uH2YJ6jpR1	tmpd_fYpy
    gpg-2xGcVB	ion_channel.tpr  openmpi-sessions-stefan@meltingpot_0  singularity   singularity-test.iDusMr		    sni-qt_keepassx_8967-rlxQtE      testcontainer.img	tmpGHNsdK

Now source the gromacs environment and run a short test

    stefan@meltingpot:/tmp$ source /usr/share/gromacs/shell-specific/GMXRC.bash
    stefan@meltingpot:/tmp$ OMP_NUM_THREADS=1 mpirun -n 2 gmx_mpi_d mdrun -s ion_channel.tpr -maxh 0.50 -noconfout -nsteps 500 -g logfile -v > mdrun.out
    (...)
    
While the test is running open up another terminal on your machine and verify using `top` that the MPI processes are actually running

    25328 stefan    39  19  534760 145088  15576 R 100,0  1,8   1:35.90 mdrun_mpi_d.ope                                                                                                                                                          
    25329 stefan    39  19  527960 139272  15252 R 100,0  1,8   1:35.90 mdrun_mpi_d.ope

After a few minutes the test should have finished and you can exit the container. Note that performance may be much worse than you're used to due to lacking AVX2 support!

    stefan@meltingpot:/tmp$ exit

## Port container to JUSTUS (or other hosts)

A few things must still be done to be able to redistribute this container:
1. Include example run script
2. Include example data file

Singularity containers have an default run script under `/singularity`
This is invoked if we execute the container like so

    sudo chmod +x testcontainer.img
    ./testcontainer.img
    
So let's create a run script

    cat <<EOF > singularity
    #!/bin/bash
    source /usr/share/gromacs/shell-specific/GMXRC.bash
    OMP_NUM_THREADS=1 mpirun -n 2 gmx_mpi_d mdrun -s /data/ion_channel.tpr -maxh 0.50 -noconfout -nsteps 500 -g logfile -v > /tmp/mdrun.out
    EOF
    cp 

and copy it into the container

    sudo singularity cp ./singularity testcontainer.img /singularity
    
Copy the data file

    sudo singularity exec -w testcontainer.img mkdir /data
    sudo singularity cp ion_channel.tpr testcontainer.img /data/ion_channel.tpr
    
Now copy this container to justus and run it there as well

    rsync -rahzvp testcontainer.img justus:
    
    ssh yourusername@justus
    ssh n0726
    ./testcontainer.img
