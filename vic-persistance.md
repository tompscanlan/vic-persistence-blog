
# Persistent Storage for vSphere Integrated Containers

[vSphere Integrated Containers (VIC)](https://www.vmware.com/products/vsphere/integrated-containers.html) is a container solution that leverages the strengths of vSphere.  Let's dive into some of the the storage concerns with docker containers and see how they are addressed in VIC.

## Docker Image Layers

Running containers are composed of layers of filesystem images applied in a stack.  The layers are created at build-time, and remain unaltered even at run-time.  While running, changes to the filesystem will be persisted to an extra layer called the container layer.  The container layer is removed when the container is removed.

For example, this docker file builds on top of the alpine-3.6 image layer.

```
FROM alpine:3.6

RUN echo -e "#!/bin/sh\ndate\nsleep 2d\ndate" >/bin/our-application
RUN chmod 755 /bin/our-application
CMD ["/bin/our-application"]
```

Building an image using this docker file results in an image with several layers that can be seen using `docker history`.

```
$ docker build -f Dockerfile.example-1 -t demo:0.1 .
Sending build context to Docker daemon  7.168kB
Step 1/4 : FROM alpine:3.6
 ---> 76da55c8019d
Step 2/4 : RUN echo -e "#!/bin/sh\ndate\nsleep 2d\ndate" >/bin/our-application
 ---> Running in dfce6e80a2fb
 ---> 9295df9995e6
Removing intermediate container dfce6e80a2fb
Step 3/4 : RUN chmod 755 /bin/our-application
 ---> Running in cdc0e6d7ba27
 ---> 1d5559a943d4
Removing intermediate container cdc0e6d7ba27
Step 4/4 : CMD /bin/our-application
 ---> Running in d44e2734bef0
 ---> 31af83e49686
Removing intermediate container d44e2734bef0
Successfully built 31af83e49686
Successfully tagged demo:0.1

$ docker history demo:0.1
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
31af83e49686        2 minutes ago       /bin/sh -c #(nop)  CMD ["/bin/our-applicat...   0B
1d5559a943d4        2 minutes ago       /bin/sh -c chmod 755 /bin/our-application       29B
9295df9995e6        2 minutes ago       /bin/sh -c echo -e "#!/bin/sh\ndate\nsleep...   29B
76da55c8019d        4 weeks ago         /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B
<missing>           4 weeks ago         /bin/sh -c #(nop) ADD file:4583e12bf5caec4...   3.97MB
```

When we run the container, an additional container layer is created that allows modification of the filesystem by the running system.  If no changes are made to the filesystem, this layer remains empty.  Let's run the image as a container, and modify it's filesystem:

```
$ docker run -d --name demo --rm  demo:0.1
$ docker exec -it demo sh
/ # ls /
bin    dev    etc    home   lib    media  mnt    proc   root   run    sbin   srv    sys    tmp    usr    var
/ # date > /demo-state
/ # ls /
bin         dev         home        media       proc        run         srv         tmp         var
demo-state  etc         lib         mnt         root        sbin        sys         usr
```
As long as the container runs, the file `demo-state` will exist and have the same contents.  Stopping and starting the container has no effect on the container layer, so `demo-state` will still exist.

If we stop and remove the container, the container layer will be removed, and our hold on the `demo-state` file.  Running a new instance of the container will have a new empty container layer.

For more details on the structure of images see https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers/


## Why aren't containers persistent?

The reason for this lack of persistence is to ensure the application we put into a container image is always the application being run.  Images are versioned so that we can be sure that two systems are running exactly the same code.  Re-running the same image will always produce the same running conditions. 

The immutability of the images results in better ability to debug, smoother deployments, and the ability to quickly replace running applications that appear to be in a bad state.

Let's flip it around; If container images were able to change, how could I be sure running a specific image today and running it tomorrow would have the same results?  How could I debug an image on my laptop, and be sure I am seeing the same code that is having a problem in QA?  If an application has persisted state in its local image, how do other instances of the application container get access to that data?

## Ok, lack of persistence is good for the container... how do I save data?

At some point, most of our applications need to leverage some data.  
How do we keep state between runs of an image?  There are a at least a few patterns:

1. replication
2. replay transactions
3. filesystem persistance

### Replication

If you can design your application to replicate data to other containers, and ensure at least one copy is always running then you're using this pattern.

An example of this patterns is running a Cassandra database cluster, where replication enables the dynamic addition or removal of nodes.  ***I wouldn't recommend doing this***, **but** if running Cassandra in containers, and being good about bootstrapping, and removing nodes, then you could run a stable database cluster with normal basic `docker run`.  The persistence is handled by storing data in the container layer.  As long as enough containers are up, persistence is maintained.

### Re-create data on loss

If you can design your application to be able to re-create any needed data, you're using this pattern.

An example of this might be a prime number finder, where if the discovered primes are lost due to the container dying, the same primes can be re-discovered in another container.

A less brain-dead example would be to leverage an external kafka service as a journal to save every transaction.  When a container dies, and is replaced, it can replay all the prior transactions until state is up-to-date.

### Persistent Filesystem

We can leverage an existing persistent filesystem that lives on the docker host inside the container.  This is a pattern most of us are familiar with, as it has been *the* way to handle data persistence since tape drives were invented.

Docker has two ways of handing a persistent filesystem in containers **bind mounts** and **volumes**.  Both of these expose a filesystem into the container from the running host.  They are very similar, but the bind mount is a bit more limited than using volumes.

#### bind mount

This is simply mounting a host filesystem file or directory into the container. This is not very different from mounting a CDROM onto a VM.  The host path may look like `/srv/dir-to-mount`, and inside the container, you may be able to access the directory at `/mnt/dir-to-mount`

#### volumes

This is slightly different from a simple bind mount.  Here, docker creates a directory that is the volume, and mounts it just as in a bind mount.  A big difference is that docker manages the lifecycle of this volume.  This is the preferred way to use persistent storage in docker.

From [Docker's volume document](https://docs.docker.com/engine/admin/volumes/volumes/):

> * Volumes are easier to back up or migrate than bind mounts.
> * You can manage volumes using Docker CLI commands or the Docker API.
> * Volumes work on both Linux and Windows containers.
> * Volumes can be more safely shared among multiple containers.
> * Volume drivers allow you to store volumes on remote hosts or cloud providers, to encrypt the contents of volumes, or to add other functionality.
> * A new volume’s contents can be pre-populated by a container.

Lets take a closer look at using volumes to persist data in VIC.

## VIC Volumes

Command line use of volumes in VIC is exactly the same as standard docker, with the benefit of the storage being backed by NFS or vSphere Storage.

### create and use volumes

Create the volume:

```
$ docker volume create  --name demo_vol
$ docker volume ls
DRIVER              VOLUME NAME
vsphere             demo_vol
```

Mount it into a container, and save some state:

```
$ docker run -it --rm -v demo_vol:/vol  busybox sh
# date >> /vol/demo-state
# exit
```

Re-use the same volume in another container:

```
$ docker run -it --rm -v demo_vol:/vol  busybox sh
# date >> /vol/demo-state
# cat /vol/demo-state
# exit
```

```
$ export DOCKER_HOST=192.168.100.138:2375
$ export DOCKER_CONTENT_TRUST_SERVER=https://registry.corp.local:4443
```

One thing to look out for, is that under native docker volumes only exist on the  host for the container.  If a host dies, the underlying storage is lost.  
A cool feature of the volume being backed by vSphere, is that a running container can be scheduled on any host that is part of the same DRS cluster, and if the chosen host goes down, the container resumes running on a different host.

Here is the initial placement of a container:
![image of initial container placement](./initial-container.png)

And after a host failure, the container is moved to a running host:
![Host failure migrates container](./migrated-container.png)


### caveats
* volumes are centralized on shared storage, either VMFS or NFS

Currently there is no support for volumes under Storage Distributed Resource Scheduler (SDRS).

Currently vSphere backed volumes are do not support being mounted on more than one container at a time, though this is supported with NFS backed volumes.
