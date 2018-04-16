# OpenStack Installation Guideline

## Requirements
Disk=20G
Memory=4G

## Turn Off Services
systemctl stop firewalld
systemctl disable firewalld
systemctl stop NetworkManager
systemctl disable NetworkManager
systemctl enable network
systemctl start network

## Disable selinux from it's config file
vi /etc/selinux/config
SELINUX=disabled
reboot
getenforce

## RHEL
sudo yum install -y https://rdoproject.org/repos/rdo-release.rpm

## CentOS Install OpenStack Package
sudo yum install -y centos-release-openstack-ocata
sudo yum update -y

## Run packstack installer
packstack --allinone --provision-demo=n --os-neutron-ovs-bridge-mappings=extnet:br-ex --os-neutron-ml2-type-drivers=xvlan,flat

## Make Interface Layer 2 Bridge Port
vi /etc/sysconfig/network-scripts/ifcfg-enpOs3

'''
TYPE=OVSPort
NAME=enpsOs3
DEVICE=enpsOs3
DEVICETYPE=ovs
OVS_BRIDGE=br-ex
ONBOOT=yes
'''

vi /etc/sysconfig/network-scripts/ifcfg-br-ex
DEVICE=br-ex
DEVICETYPE=ovs
TYPE=OVSBridge
BOOTPROTO=static
IPADDR=192.168.0.108
NETMASK=255.255.255.0
GATEWAY=192.168.0.1
IPV4_FAILURE_FATAL=no
IPV6INIT=no
DNS1=8.8.8.8
ONBOOT=yes

service network restart

## OpenStack admin privileges
source keystonerc_admin

## Create Provider Network
Create provider network instances to communicate with outside world
neutron net-create external_network --provider:network_type flat --provider:physical_network extnet --router:external

## Create Subnet attached to provider network
neutron subnet-create --name public_subnet --enable-dhcp=False --allocation-pool start=192.168.0.100,end=192.168.0.120 --gateway=192.168.0.1 external_network=192.168.0.0/24


## Verify OpenStack Installation
yum install -y openstack-rally
rally-manage db recreate
source keystonerc_admin
rally deployment create --fromenv --name=existing

## Horizon Dashboard
go to http://192.168.0.108
admin
passwd:e8b28cc0115b400b

## List Open Stack Service
cd /etc/systemd/system/multi-user.target.wants/ && ls *.service

## Create Image in Glance
https://docs.openstack.org/image-guide/obtain-images.html
User cirros for testing lightweight 13Mb 

Instance(Nova):
- Image (Glance)
- Network (Neutron)
- Size of the instance (Nova)
- Security Settings:
    - ACLs(Neutron)
    - Key pair (Nova)
- Persitent storage (Cinder)

## Database clustering & Scaling
galera cluster
message queue clustering (rappidmq,qpid,zeromq)
built-in scaling: neutron, swift proxy
API endpoint scaling: load balancer
pacemaker

## Multi Node Design & Scaling
One network node with multiple compute nodes
Non compute node based shared file system:
- Disks storing running instances are hosted in servers outside of the compute nodes
- Compute hosts are stateless. In case of node failure, instances are easily recoverable

On compute node storage - shared file system:
- Each compute node is specified with a significant amount of disk space
- A distributed file system ties the disks from each compute node into a single mount
- Data locality is lost
- Complicated instance recovery

On compute node storage - nonshared file system
- Local disks on compute nodes used
- Heavy I/O usage on one compute node does not affect instances on other nodes
- When you lose a node, data on it gets losts as well
- Instance migration is complicated



## Controller Node
- neutron-server
- nova-api
- nova-scheduler
- nova-conductor
- keystone-all
- glance-api
- glance-registry
- cinder-api
- cinder-scheduler
- message-queue
- database
