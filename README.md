# Using Modern File Systems to Create Efficient Snapshots Effortlesly

This article will demonstrate how easy it is, nowadays, to take an efficient snapshot of anything stored on a _modern_ file system. That includes data stored inside a database.

## **What Is a Snapshot**

A snapshot is an atomic copy of a dataset made in order to freeze it at a given moment. It is  useful in many situations. For example, it makes it possible to
*   make a backup
*   implement the[ read-copy-update (RCU)](https://en.wikipedia.org/wiki/Read-copy-update) algorithm
*   implement an _undo_ feature
*   etc.

## **How to Take Snapshots**

### **Outside the Context of a File System**

Let says you are writing a program that needs to copy some dataset.

#### **The Naive Way**

Snapshots can be made by copying each value of the dataset while temporarily preventing updates from taking place. However,
*   It wastes space and time because it completely duplicates the dataset, regardless of the fact that some members will be identical between copies.
*   It the dataset doesn't occupy a continuous space in memory, the copy will need to be re-implement for different kinds of datasets, which is new code to be developed, tested and maintained.

#### **The Functional Programming or Immutable Way**

At the opposite end of the spectrum, there are immutable data structures. Since their content cannot be modified, acquiring references is all that is required to "copy" them. Well designed data structures are organized in clever ways that make evolving datasets share most of their memory. Have a look at this[ thesis](https://homepages.cwi.nl/~jurgenv/papers/PhDThesis_Michael_Steindorfer.pdf), for example, where the author addresses the performance of immutable data structures in Java.

#### **Copy on Write**

Copy on Write (CoW) is an old concept. In the Unix world, it has been used for sharing pages of RAM between multiple processes[ since at least the mid 90s](https://obvious.services.net/2011/01/history-of-copy-on-write-memory.html), although the history of the concept goes back to the early 70s. However, the concept is used deep inside the operating system and is not apparent to most users. Just like with the immutable approach, all that is needed when copying a dataset is a reference to it. However, when a modification is applied, part of the dataset is duplicated.

### **In the Context of a File System**

File systems allow one storage space (a disk) to appear as many different storage spaces (files) accessible by names organized into a hierarchy (folders). To optimize memory allocation, disks are divided into blocks. The size of blocks is determined at the creation of file systems and, at the lowest level, they are the smallest element through which data is manipulated.

#### **The Naive Way**

Copying a folder recursively, while preventing concurrent modifications, is always an option on any file system. As said before, it is terribly inefficient.

#### **Hard Links**

We have been stuck with basic file systems for a while. In the Unix world, the only way for two files to share the same data on the disk was through hard links, which is when a file has two different names. Only when all the names have been deleted is the space on the disk marked as free. That possibility did not even exist in the Windows world since the advent of the NTFS file system in 1993. Hard links are sufficient to make efficient snapshots of folders, _provided the content of the files themselves do not change_.

##### **How-To Do It**

On Linux, the command
```
$ cp -al
```
can copy huge folders very quickly, and without taking a lot of extra space on the disk. It is shocking how fast it is when it is used for the first time.

#### **Copy on Write**

CoW allows files to share blocks. It is about time we got it and it is the main feature of the **BTRFS** file system. Development started in 2007, but adoption has been slow.

Nevertheless, it has been declared stable in the Linux Kernel since 2013 and it is the default file system of the[ SUSE Linux Enterprise Server](https://en.wikipedia.org/wiki/SUSE_Linux_Enterprise_Server) distribution since 2015. It is also supported by Oracle Linux since 2012.

But, what is really exciting is that CoW has been added to **XFS**, a fast, mature ([it dates back to 1993](https://en.wikipedia.org/wiki/XFS)) file system. The file system is the default of RHEL and the feature is[ available in RHEL 8](https://www.redhat.com/en/about/videos/rhel-8-beta-xfs-copy-write-data-extents).

##### **How-To Do It**

On Linux provided you use one of the two file systems mentioned above, the command
```
$ cp -a --reflink=always
```
is all that is needed. The file system can also be configured so that CoW is the default behavior. In that case, copying a file or a folder _always_ creates an efficient snapshot !


## **How It Applies To Databases**

Well, database storage engines are designed to be fast and to play well with the file system. Changes to the data will be kept to a minimum, and data is aligned on blocks. That means that a snapshot of database files using CoW should work very well and that most of the memory will be shared between the two databases, even after resulting databases are modified independently.

### **Step By Step Instructions**

We will use Docker to test this with a Postgresql image and an XFS volume mapped to the directory where Postgresql stores its data. The exact way to duplicate a database at the file system level will be different for other database vendors, but in all cases the process will be straightforward.

#### **Get Docker**

First, install docker if you don't have it already. It is not going to help you if your system doesn’t support XFS, but it will make installing, starting and stopping PostgreSQL much easier. It will also allow us to clean everything once we are done. On Ubuntu, enter this command:

```
$ sudo apt update
$ sudo apt install docker.io
$ sudo usermod -aG docker `whoami`
```

#### **Create an XFS File System**

If you don't have an XFS file system, create the file system on a file and mount it using a loop device.

```
$ sudo dd if=/dev/zero of=/xfs.1G bs=1M count=1K

$ sudo mkfs.xfs /xfs.1G
meta-data=/xfs.1G         	isize=512    agcount=4, agsize=65536 blks
     	=                   	sectsz=512   attr=2, projid32bit=1
     	=                   	crc=1        finobt=1, sparse=1, rmapbt=0
     	=                   	reflink=1
data 	=                   	bsize=4096   blocks=262144, imaxpct=25
     	=                   	sunit=0      swidth=0 blks
naming   =version 2          bsize=4096   ascii-ci=0, ftype=1
log  	=internal log       	bsize=4096   blocks=2560, version=2
     	=                   	sectsz=512   sunit=0 blks, lazy-count=1
realtime =none               extsz=4096   blocks=0, rtextents=0
```
Make sure that **crc=1** and **reflink=1** in the result of the command as we depend on those features. They should be enabled by default. Otherwise, you might be using a system that is a bit too old.

```
$ LOOP_DEVICE=`sudo losetup --show -fP /xfs.1G`
```

#### **Create a Docker Volume**

Use your brand new device to create a docker volume.

```
$ docker volume create --driver local --opt type=xfs --opt device=$LOOP_DEVICE \
  --name postgres-data-dir
postgres-data-dir
```

#### **Run a Container**

This command will download and run a postgres image with your XFS file system mounted at /var/lib/postgresql/data
```
$ docker run -d \
  -e POSTGRES_HOST_AUTH_METHOD=trust \
  -v postgres-data-dir:/var/lib/postgresql/data \
  --name cow \
  postgres
```
The container should now be running. Open a terminal in the the newly created container using the command bellow:
```
$ docker container exec -it `docker container ls -q --filter='name=cow'` \
  su postgres -s /bin/bash
```
We are running bash as user “postgres” as it simplifies the connection to the database. 

**Play With PostgreSQL**

Now, check that the XFS volume is correctly mounted:
```
$ df -hT | grep xfs
/dev/loop0 	xfs 	1014M   80M  935M   8% /var/lib/postgresql/data
```

As you can see, we are already using about 80M of the space.
```
$ du -sh /var/lib/postgresql/data/
40M    /var/lib/postgresql/data/
```

About 40M of the space used is used by files and folders (including their file system metadata). The other 40M is used by file system metadata that is not directly related to those files and folders. It is just a coincidence that postgres uses the same amount of data than the filesystem metadata.

Now, let's create a new database and "fill" it.
```
$ psql

postgres=# create database hello;
CREATE DATABASE

postgres=# select oid from pg_database where datname='hello';
  oid  
-------
 16384
(1 row)


postgres=# \c hello
You are now connected to database "hello" as user "postgres".

hello=# create table some_table(x int);
CREATE TABLE

hello=# \timing on
Timing is on.

hello=# insert into some_table select x from generate_series(1,1048576) t(x);
INSERT 0 1048576
Time: 1146.121 ms (00:01.146)

hello=# \q
```
So, it took us about 1.15s to insert 1M rows into the database. Let's have a look at how much space it takes.
```
$ df -hT | grep xfs
/dev/loop0 	xfs 	1014M  188M  827M  19% /var/lib/postgresql/data
```
At first, it looks like our database takes 108 Mb on the filesystem. However, some of this space has been taken by postgres for data that is not directly attributable to the database that we created.
```
$ du -sh /var/lib/postgresql/data/base/16384/
45M    /var/lib/postgresql/data/base/16384/
```
In fact, it takes 45 Mb of space on the disk. Let’s go back to postgres and copy the database using the standard postgres way.
```
$ psql

postgres=# \timing on
Timing is on.

postgres=# create database world template hello;
CREATE DATABASE
Time: 579.761 ms

postgres=# \timing off
Timing is off.

postgres=# select oid from pg_database where datname='world';
  oid  
-------
 16388
(1 row)


postgres=# \q

$ df -hT | grep xfs
/dev/loop0 	xfs 	1014M  233M  782M  23% /var/lib/postgresql/data

$ du -sh /var/lib/postgresql/data/base/16388
45M    /var/lib/postgresql/data/base/16388
```
This time, the space on the filesystem took only 45 Mb on the filesystem, without any other increase in size. The important part is that our database and the copy both take 45 Mb of space. Now, let’s try to clone the database in a more efficient manner. Let’s first delete the copy we made.
```
$ psql

postgres=# drop database world;
DROP DATABASE

postgres=# \q

$ df -hT | grep xfs
/dev/loop0 	xfs 	1014M  188M  827M  19% /var/lib/postgresql/data
```
We will create an empty database, just so postgresql creates a folder for it and an entry in pg_database. Then, we will stop the service (and, incidentally, the container), copy the database using <code>cp --reflink=always</code>, and restart the service. The new database should be a clone of the first one.
```
$ psql

postgres=# \timing on
Timing is on.

postgres=# create database world;
CREATE DATABASE
Time: 370.861 ms

postgres=# \timing off
Timing is off.

postgres=# select oid from pg_database where datname='world';
  oid  
-------
 16389
(1 row)


postgres=# \q

$ exit
```

**Perform an Efficient Snapshot**

We are now “out” of the container and back to the host machine. First, let’s stop the container.
```
$ docker container stop cow
```
And then perform the copy.
```
$ mkdir tmp

$ sudo mount $LOOP_DEVICE tmp

$ sudo rm -fr tmp/base/16389 # remove empty database files

$ time sudo cp -a --reflink=always tmp/base/16384 tmp/base/16389 # clone database files

real    0m0,014s
user    0m0,002s
sys     0m0,011s

$ sudo umount tmp
```
It was pretty fast, right ? Now let’s start the container again, this time in the background.
```
$ time docker start cow
cow
real    0m0,411s
user    0m0,025s
sys     0m0,015s
```
And let’s connect again to it.
```
$ docker container exec -it `docker container ls -q --filter='name=cow'` \
  su postgres -s /bin/bash

$ df -hT | grep xfs
/dev/loop0 	xfs 	1014M  189M  826M  19% /var/lib/postgresql/data
```
Pretty good, cloning the database took almost no extra space. But does it work ?
```
$ psql world

world=# select count(*) from some_table;
  count  
---------
 1048576
(1 row)
```
Yes, it does. Let’s add a row.
```
world=# insert into some_table values (123456789);
INSERT 0 1

world=# select count(*) from some_table;
  count  
---------
 1048577
(1 row)

world=# \c hello

hello=# select count(*) from some_table;
  count  
---------
 1048576
(1 row)

hello=# \q
```
Good, adding a row on the copy has no impact on the original. Let’s look at space usage.
```
$ df -hT | grep xfs
/dev/loop0 	xfs 	1014M  189M  826M  19% /var/lib/postgresql/data
```
There is no significant additional space usage, which means that only the modified block on the file system has been copied.

**Clean everything**

```
$ exit
```
You should now be back on the host machine.
```
$ docker stop cow
$ docker container rm cow
$ docker volume rm postgres-data-dir
$ rmdir tmp
$ sudo rm /xfs.1G
```

## **How It Applies to Web Services**

Using the technique above, one can implement the RCU algorithm on any web service that serves information from a database. When a client wants to be able read data in isolation from concurrent modifications, it asks for a session token. Database is then copied by the web service using the token as its name. Yes, that means restarting the database service. Finally, when the service receives a request with a session token attached, it can simply use a connection to the copied database instead of the main one.
