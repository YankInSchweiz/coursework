# Homework: Part 1 - Installing GPFS FPO

## Overview

These instructions are a subset of the official instructions linked to from here: [IBM Spectrum Scale Resources - GPFS](http://www-03.ibm.com/systems/storage/spectrum/scale/resources.html).


We will install GPFS FPO with no replication (replication=1) and local write affinity.  This means that if you are on one of the nodes and are writing a file in GPFS, the file will end up on your local node unless your local node is out of space.

A. __Get three virtual servers provisioned__, 2 vCPUs, 4G RAM, UBUNTU\_16\_64, __two local disks__ 25G each, in San Jose. __Make sure__ you attach a keypair.  Pick intuitive names such as gpfs1, gpfs2, gpfs3.  Note their internal (10.x.x.x) ip addresses.

B. __Set up each one of your nodes as follows:__

Add to /root/.bash\_profile the following line in the end:

    export PATH=$PATH:/usr/lpp/mmfs/bin

Make sure the nodes can talk to each other without a password.  When you created the VMs, you specified a keypair.  Copy it to /root/.ssh/id\_rsa (copy paste or scp works).  Set its permissions:

    chmod 600 /root/.ssh/id_rsa

Set up the hosts file (/etc/hosts) for your cluster by adding the __PRIVATE__ IP addresses you noted earlier and names for each node in the cluster.  __Also__ you should remove the entry containing the fully qualified node name for your headnode / gpfs1.sftlyr.ws (otherwise it will trip up some of the GPFS tools since it likely does not resolve). For instance, your hosts file might look like this:

    127.0.0.1 		localhost.localdomain localhost
    10.122.6.68		gpfs1
    10.122.6.70		gpfs2
    10.122.6.71		gpfs3

Create a nodefile.  Edit /root/nodefile and add the names of your nodes.  This is a very simple example with just one quorum node:

    gpfs1:quorum:
    gpfs2::
    gpfs3::

C. __Install and configure GPFS FPO on each node:__

You will need to obtain the GPFS tarball from us.  Check the wall for a post about where to obtain the URL to download the file  Spectrum\_Scale\_ADV\_501\_x86\_64\_LNX.tar.

As root on your node, download this tarball into /root, then unpack and install:
```
apt-get update
apt-get install ksh binutils libaio1 g++ make m4
cd
tar -xvf Spectrum_Scale_ADV_501_x86_64_LNX.tar
```
Then install GPFS with:
```
./Spectrum_Scale_Advanced-5.0.1.0-x86_64-Linux-install --silent
cd /usr/lpp/mmfs/5.0.1.0/
dpkg -i *.deb
/usr/lpp/mmfs/bin/mmbuildgpl
```

D. __Create the cluster.  Do these steps only on one node (gpfs1 in my example).__

Now we are ready to create our cluster.  I named mine \[ -C\] dima .. Make sure you pass the correct node file to the --N command.

    mmcrcluster -C dima -p gpfs1 -s gpfs2 -R /usr/bin/scp -r /usr/bin/ssh -N /root/nodefile

Now, you must accept the license:

    mmchlicense server -N all
    # (say yes)

Now, start GPFS:

    mmstartup -a

All nodes should be up ("GPFS state" column shows "active"):

    mmgetstate -a

Nodes may reflect "arbitrating" state briefly before "active".  If one or more nodes are down, you will need to go back and see what you might have missed.  The main GPFS log file is `/var/adm/ras/mmfs.log.latest`; look for errors there.

You could get more details on your cluster:

    mmlscluster

Now we need to define our disks. Do this to print the paths and sizes of disks on your machine:

    fdisk -l

Note the names of your 25G disks. Here's what I see:

    [root@gpfs1 ras]# fdisk -l |grep Disk |grep bytes
    Disk /dev/xvdc: 26.8 GB, 26843545600 bytes
    Disk /dev/xvda: 26.8 GB, 26843545600 bytes
    Disk /dev/xvdb: 2147 MB, 2147483648 bytes

Now inspect the mount location of the root filesystem on your boxes:

    [root@gpfs1 ras]# mount | grep ' \/ '
    /dev/xvda2 on / type ext3 (rw,noatime)

Disk /dev/xvda (partition 2) is where my operating system is installed, so I'm going to leave it alone.  In my case, __xvdc__ is my second 25 disk.  In your case, it could be /dev/xvdb, so __please be careful here__.  Assuming your second disk is `/dev/xvdc` then add these lines to `/root/diskfile.fpo`:

    %pool:
    pool=system
    allowWriteAffinity=yes
    writeAffinityDepth=1

    %nsd:
    device=/dev/xvdc
    servers=gpfs1
    usage=dataAndMetadata
    pool=system
    failureGroup=1

    %nsd:
    device=/dev/xvdc
    servers=gpfs2
    usage=dataAndMetadata
    pool=system
    failureGroup=2

    %nsd:
    device=/dev/xvdc
    servers=gpfs3
    usage=dataAndMetadata
    pool=system
    failureGroup=3

Now run:

    mmcrnsd -F /root/diskfile.fpo

You should see your disks now:

    mmlsnsd -m

Let’s create the file system.  We are using the replication factor 1 for the data:

    mmcrfs gpfsfpo -F /root/diskfile.fpo -A yes -Q no -r 1 -R 1

Let’s check that the file system is created:

    mmlsfs all

Mounting the distributed FS (be sure to pass -a so that the filesystem is mounted on all nodes):

    mmmount all -a

All done.  Now you should be able to go to the mounted FS:

    cd /gpfs/gpfsfpo

.. and see that there's 75 G there:

    [root@gpfs1 gpfsfpo]# df -h .
    Filesystem      Size  Used Avail Use% Mounted on
    /dev/gpfsfpo     75G  678M   75G   1% /gpfs/gpfsfpo

Make sure you can write, e.g.

    touch aa

If the file was created, you are all set:

    ls -l /gpfs/gpfsfpo
    ssh gpfs2 'ls -l /gpfs/gpfsfpo'
    ssh gpfs3 'ls -l /gpfs/gpfsfpo'

Proceed to [Part 2 - The Mumbler](../the_mumbler).
