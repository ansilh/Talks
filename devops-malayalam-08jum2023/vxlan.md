# VXLAN demo

## Pre-requisites

- Create two VMs in VirtualBox.
- Assigne one host-only network interfaces (NICs) and one NAT interface.
- Assign IP to first NIC.
- Install Rocky Linux 9 (Server without UI) on both VMs.
- Set hostnames as node01 and node02
- Make sure internet is working on VMs (we have to download few files along the way).
- Assuming that node01 IP is 192.168.56.21 and node02 IP is 192.168.56.22

## Etcd installation and setup on node01

- Download etcd binary
```bash
ETCD_VER=v3.5.0
DOWNLOAD_URL=https://storage.googleapis.com/etcd
curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
mkdir /opt/etcd
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /opt/etcd --strip-components=1
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
```
- Start etcd 
```bash
nohup /opt/etcd/etcd -listen-client-urls=http://192.168.56.21:2379 --advertise-client-urls=http://192.168.56.21:2379 >/var/log/etcd.log 2>&1 &
/opt/etcd/etcdctl --endpoints http://192.168.56.21:2379 member list -w table
```
## Falnneld setup on both node01 and node02

- Need to execute below from node01
```bash
/opt/etcd/etcdctl --endpoints http://192.168.56.21:2379 put /coreos.com/network/config '{ "Network": "10.5.0.0/16", "Backend": {"Type": "vxlan"}}'
/opt/etcd/etcdctl --endpoints http://192.168.56.21:2379 get /coreos.com/network/config 
```
- Both node01 and node02
```bash
mkdir /opt/flannel
curl -L https://github.com/ansilh/Talks/raw/main/cncf-kochi-29apr2023/bins/flanneld -o /opt/flannel/flanneld && chmod +x /opt/flannel/flanneld
```
```bash
# node01
nohup /opt/flannel/flanneld --etcd-endpoints=http://192.168.56.21:2379 --public-ip=192.168.56.21  >/var/log/flannel.log 2>&1 &
# node02 
nohup /opt/flannel/flanneld --etcd-endpoints=http://192.168.56.21:2379 --public-ip=192.168.56.22  >/var/log/flannel.log 2>&1 &
```

- Create container root file system for container no both nodes
```bash
mkdir -p container-demo
cd container-demo
```
- Download busybox
```bash
curl -LO https://github.com/ansilh/Talks/raw/main/cncf-kochi-29apr2023/bins/busybox
```
- Create symlink for bins
```bash
chmod +x busybox
for i in $(./busybox --list)
do
   ln -s /busybox bin/$i
done
cd ..
```

- Start container in differnt namespaces on both nodes
```bash
PATH=${PATH}:/bin unshare --mount --uts --ipc --net --pid --fork --user --map-root-user --mount-proc chroot container-demo /bin/sh
```
- Create veth pair on both nodes
```bash
ip link add vethlocal type veth  peer name vethNS
```
- Change MTU to 1440 on both nodes
```bash
ip link set dev vethNS mtu 1450
ip link set dev vethlocal mtu 1450
```

- Assign IP address (vxlan) and make interfaces up on both nodes
```bash
# on both nodes
awk -F "=" '$1 ~ /^FLANNEL_SUBNET/{print $2}' /var/run/flannel/subnet.env | awk -F "." '{print $1"."$2"."$3"."10}'
```
```bash
#  Replace {IP} with the IP got from above command
ip addr add {IP}/24 dev vethNS
ip link set lo up
ip link set dev vethNS up
```
```bash
ip link set dev vethlocal up
```
- Create bridge , add child devs and assign IP on both nodes
```bash

nmcli con add ifname cbr0 type bridge con-name cbr0 
nmcli con add type bridge-salve ifname vethlocal master cbr0
nmcli con mod cbr0 bridge.stp no

cat /var/run/flannel/subnet.env
# Replace {SUBNET} with the IP we got from above file.

nmcli con mod cbr0 ipv4.address {SUBNET}/24
nmcli con mod cbr0 ipv4.method manual
nmcli con up bridge-slave-vethlocal
nmcli con up cbr0
nmcli con show --active

```
- Add default route in both containers based on the {SUBNET} we get from each nodes
```bash
cat /var/run/flannel/subnet.env
# Replace {SUBNET} with the IP we got from above file.
ip route add default via {SUBNET}
```
- Enable ip forwarding on both nodes
```bash
sysctl -w net.ipv4.ip_forward=1
```

- How to check bridge forwarding database

```
bridge fdb show
```

- How to check ARP entry created by flannel

```
ip neigh show dev flannel.1
```
- How to check the vxlan interface details
```
ip -d link 
```