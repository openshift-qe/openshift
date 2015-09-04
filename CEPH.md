## Setting up Ceph in a Single Container

Here is an example of how to set up and run ceph-rbd in a single container. .

### Environment:
The enviromnent used for all of the examples in this repo is described [here](../ENV.md).

### Ceph:
A Fedora 21 VM was used to run containerized ceph. It’s important to create an additional disk on your ceph VM in order to map the ceph image (not to be confused with a docker image) to this extra disk device. Create an extra 8GB disk which shows up as */dev/vdb*. Install ceph-common (client libraries) so that the OSE pod running mysql can do the ceph RBD mount .

```
$ yum install -y ceph-common
 
# for now there is a ceph packaging bug where the ceph-rbdnamer
# code is missing from ceph-common, therefore ceph also needs to
# be installed on the node
$ yum install -y ceph
```

Pull ceph-docker and run all of the ceph processes inside a single Docker container:

```
$ docker pull ceph/demo
#stash the image locally. At some point I will need to use the ceph/daemon image to
#create the monitor, osd, and rgw as separate containers
 
$ docker run --net=host -v /etc/ceph:/etc/ceph -v /var/lib/ceph:/var/lib/ceph \
  -e MON_IP=192.168.122.133-e CEPH_NETWORK=192.168.122.0/24 ceph/demo
 
$ ps -e | grep ceph
29242 ?        00:00:30 ceph
29282 ?        00:02:48 ceph-mon
29452 ?        00:14:00 ceph-osd
29664 ?        00:03:01 ceph-mds
29725 ?        00:00:00 ceph-rest-api defunct
29993 ?        00:00:00 ceph-msgr
29997 ?        00:00:00 ceph-watch-noti
```

192.168.122.133 is the eth0 IP address for my F21 VM. This container runs in the foreground and logs to the terminal from which it was launched. If you get stuck on this step be sure to rm -rf /etc/ceph/* and potentially rm -rf /var/lib/ceph/* before retrying.

From a different terminal window on the same F21 ceph VM, create an rbd image (not be be confused with container images) and map it to a /dev/rbdX device file:

```
# Note: sometimes the command "modprobe rbd" is required
$ rbd create foo -s 1024 #creates a rbd image named "foo", 1GB in size
$ rbd map foo
 
$ rbd showmapped   #get the rbd dev, here it's /dev/rbd0
id pool image snap device 
0 rbd foo - /dev/rbd0
 
# create a file system on the block dev
$ mkfs.ext4 /dev/rbd0 #same rbdX as above
 
$ rbd --image foo -p rbd info
     rbd image 'foo':
         size 1024 MB in 256 objects
         order 22 (4096 kB objects)
         block_name_prefix: rb.0.100f.74b0dc51
         format: 1
```