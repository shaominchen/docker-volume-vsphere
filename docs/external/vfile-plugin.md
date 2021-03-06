---
title: vFile volume plugin for Docker

---
## Overview
Depending on the underlying block storage system, it might not be possible to access the same
persistent volume across different hosts/nodes simultanously.
For example, currently users cannot mount the same persistent volume which is created through
vSphere Docker Volume Service (vDVS) on containers running on two different hosts at the same time.

This can be solved through distributed file systems, such as NFS, Ceph, Gluster, etc.
However, setting up and maintaining enterprise storage offerings for Cloud Native usecases is not a trivial work.
Furthermore, users can face more challenges in order to achieve high availability, scalability, and load balancing.

__vFile volume plugin for Docker__ provides simultaneous persistent volume access between hosts in the
same Docker Swarm cluster for the base volume plugin service such as vDVS, with zero configuration effort,
along with high availability, scalability, and load balancing support.

## Detailed documentation
Detailed documentation can be found on our [GitHub Documentation Page](http://vmware.github.io/docker-volume-vsphere/documentation/).

## Prerequisites
* Docker version: 17.06.0 or newer
* Base docker volume plugin: [vSphere Docker Volume Service](https://github.com/vmware/docker-volume-vsphere)
* All hosts running in [Swarm mode](https://docs.docker.com/engine/swarm/swarm-tutorial/)

## Installation
The recommended way to install vFile plugin is from docker cli:

```
docker plugin install --grant-all-permissions --alias vfile vmware/vfile:latest
```

Note: please make sure the base volume plugin is already installed!

## Usage examples

### Steps for create/use/delete a vFile volume

#### Creating a persistent volume from vFile plugin

```
$ docker volume create --driver=vfile --name=SharedVol -o size=10gb
$ docker volume ls
$ docker volume inspect SharedVol
```

Options for creation will be the same for the base volume plugin.
Please refer to the base volume plugin for more options.
Note: vFile volume plugin doesn't support filesystem type options.
Note: The valid volume name can only be ```[a-zA-Z0-9][a-zA-Z0-9-]*[a-zA-Z0-9]```.

#### Mounting this volume to a container running on the first host

```
# ssh to node1
$ docker run --rm -it -v SharedVol:/mnt/myvol --name busybox-on-node1 busybox
/ # cd /mnt/myvol
# read/write data into mounted vFile volume
```

#### Mounting this volume to a container running on the second host

```
# ssh to node2
$ docker run --rm -it -v SharedVol:/mnt/myvol --name busybox-on-node2 busybox
/ # cd /mnt/myvol
# read/write data from mounted vFile volume
```

#### Stopping the two containers on each host

```
# docker stop busybox-on-node1
# docker stop busybox-on-node2
```

#### Removing the vFile volume

```
$ docker volume rm SharedVol
```

## Configuration
### Options for vFile plugin
Users can choose the base volume plugin for vFile plugin, by setting configuration during install process.
<!---
* Through CLI flag can only be done through non-managed plugin.
--->

* Default config file location: `/etc/vfile.conf`.
* Default base volume plugin: vSphere Docker Volume Service
* Sample config file:

```
{
        "InternalDriver": "vsphere"
}
```

The user can override the default configuration by providing a different configuration file,
via the `--config` option, specifying the full path of the file.

### Options for logging
* Default log location: `/var/log/vfile.log`.
* Logs retention, size for rotation and log location can be set in the config file too:

```
 {
	"MaxLogAgeDays": 28,
	"MaxLogSizeMb": 100,
	"LogPath": "/var/log/vfile.log"
}
```

## Q&A

### How to install and use the driver?
Please see README.md in the for the release by clicking on the tag for the release. Example: TBD.

### Can I use another base volume plugin other than vDVS?
Currently vFile volume service is only developed and tested with vDVS as the base volume plugin.

### I got "Operation now in progress" error when mounting a vFile volume to a container.
Please make sure the routing mesh of Docker Swarm cluster is working properly.
Use the following way to verify:

```
docker service create --replicas 1 -p 8080:80 --name web nginx
```

Then on each host, make sure there is valid return of `curl 127.0.0.1:8080`.
`Connection refused` error means the routing mesh of this Docker Swarm cluster is broken and needs to be fixed.

### Mounting volume failed and I saw "name must be valid as a DNS name component" error in the log when mounting a vFile volume to a container.
When you see somthing like the following in the log

```
2017-08-24 11:57:16.436786459 -0700 PDT [WARNING] Failed to create file server for volume space vol7. Reason: Error response from daemon: {"message":"rpc error: code = 3 desc = name must be valid as a DNS name component"}
```

Please make sure the volume you used is a valid volume name. A valid volume name consists of ```[a-zA-Z0-9][a-zA-Z0-9-]*[a-zA-Z0-9]```.

### I got " VolumeDriver.Mount: Failed to blocking wait for Mounted state. Error: Timeout reached; BlockingWait is not complete.." when mounting a volume.
We see this issue only on platform where the space is low. When available disk space are low, Docker Swarm service may take longer to start a service. Generally it's better free up some disk space. You can also try to increase the service start timeout value, controlled by ```VFILE_TIMEOUT_IN_SECOND``` env variable:

```
docker plugin install --grant-all-permissions --alias vfile vmware/vfile:latest VFILE_TIMEOUT_IN_SECOND=90
```

This will increase timeout to 90 sec, from default of 30 sec.


### I got "docker volume ls" operation very slow and "docker volume rm/create" a vFile volume hang forever
Please check the log at `/var/log/vfile.log` and look up if there are error message about swarm status as follow:
`The swarm does not have a leader. It's possible that too few managers are online. Make sure more than half of the managers are online.`

This error message indicates the swarm cluster is not in a healthy status.

Please follow the following [recommendations for the Swarm manager nodes setup](https://docs.docker.com/engine/swarm/how-swarm-mode-works/nodes/#manager-nodes):
    1. Run the swarm cluster with a single manager only for testing purpose.
    2. An `N` manager cluster will tolerate the loss of at most `(N-1)/2` managers.
    3. Docker recommends a maximum of seven manager nodes for a swarm.