
# Bridging demo

## Pre-requisites

- Create two VMs in VirtualBox.
- Assigne two host-only network interfaces (NICs) and one NAT interface.
- Assign IP to first NIC and keep the second one free.
- Set promescuse mode to "Allow All" on second NIC.
- Install Rocky Linux 9 (Server without UI) on both VMs.
- Set hostnames as node01 and node02.
- Make sure internet is working on VMs (we have to download few files along the way).

## Create container using busybox


- Create container root file system on both nodes

```bash
mkdir -p container-demo/{bin,proc,sys,tmp}
cd container-demo
```

- Download busybox on both nodes
```bash
curl -LO https://raw.githubusercontent.com/ansilh/container-from-scratch/main/bins/busybox
```

- Create symlink for bins on both nodes
```bash
chmod +x busybox
for i in $(./busybox --list)
do
   ln -s /busybox bin/$i
done
cd ..
```


- Start container in differnt namespaces 

```bash
PATH=${PATH}:/bin unshare --mount --uts --ipc --net --pid --fork --user --map-root-user --mount-proc chroot container-demo /bin/sh
```
- Create veth pair
```bash
ip link add vethlocal type veth  peer name vethNS
```

- Add one device to a container

```bash
ps -ef |grep '/bin/sh'
ip link set vethNS netns 1415
```

- Assign IP address
```bash
# inside container
ip addr add 10.5.19.10/24 dev vethNS
ip link set lo up
ip link set dev vethNS up

# Inside host
ip link set dev vethlocal up
```
Follow above steps on second container, but set IP `10.5.19.11`

- Create Bridge
```bash
# Inside host
cd ~/container-demo/
./busybox brctl addbr cbr0
./busybox brctl addif cbr0 enp0s8
./busybox brctl show
./busybox brctl addif cbr0 vethlocal
ip link set dev cbr0 up
```
> Note:- Enable promescous mode
- Test the container network by pinging the container IP from another container