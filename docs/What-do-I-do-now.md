# What do I do now?

Now that you have a fully functional OpenStack Cloud, you need to do a couple of things to get your cloud fully operational and usable for applications like Atmosphere.

## Ensure Infrastructure Requirements

### Cisco Switch/Router Requirements

**IMPORTANT** If one is using Cisco gear with this deployment setup, it is an absolute requirement to have a IGMP Snooping Querier on the **SAME** subnet as the tunneling VLAN network.  For example, if using the default range of IP addresses defined in the OpenStack Ansible deployment, i.e. `tunnel: 172.29.240.0/22`, one **MUST** have an `IGMP Snooping Querier` within that range, or `Multicast` traffic for `HA L3 Neutron agents` will **NOT** work!

To prevent this, it is suggested that one set their `IGMP Snooping Querier` IP to `172.29.243.254`.  Just note that this address only applies if using the above tunnel block of IP addresses.

## Setting up your cloud for general use

### Creating OpenStack Glance Images

Follow instructions below from the source here: <http://docs.openstack.org/liberty/install-guide-ubuntu/glance-verify.html>

1. SSH into the any of the `Infrastructure Control Plane Hosts` or `ICPH`, attach to the `utility` container, and run the following commands to create images:

	```
	lxc-attach -n infra<ICPH-#><LITERAL-TAB>util<LITERAL-TAB>

	source openrc

	wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img

	glance image-create --name "cirros" --file cirros-0.3.4-x86_64-disk.img --disk-format qcow2 --container-format bare --visibility public --progress

	glance image-list
	
	# above command should display an image called "cirros"
	
	wget https://cloud-images.ubuntu.com/trusty/current/trusty-server-cloudimg-amd64-disk1.img
	
	glance image-create --name "ubuntu-14.04" --file trusty-server-cloudimg-amd64-disk1.img --disk-format qcow2 --container-format bare --visibility public --progress
	
	wget http://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud-1503.qcow2
	
	glance image-create --name "centos-7" --file CentOS-7-x86_64-GenericCloud-1503.qcow2 --disk-format qcow2 --container-format bare --visibility public --progress
	```

1. Glance should now report at least one of the images downloaded above

	```
	root@infra1_utility_container-<ID>:~# glance image-list
+--------------------------------------+--------------+
| ID                                   | Name         |
+--------------------------------------+--------------+
| random-id-1-ahgh8caetha9sahc9bu5OJ6g | centos-7     |
| random-id-2-ahgh8caetha9sahc9bu5OJ6g | cirros       |
| random-id-3-ahgh8caetha9sahc9bu5OJ6g | ubuntu-14.04 |
+--------------------------------------+--------------+
	```

### Create OpenStack Neutron Networks

Neutron is configured in OSA to use a HA VRRP L3 agent implemented using Linuxbridge.  In order to get `neutron` working for instance launches so that one can login via a public IP address, some networking is required to get things running.

1. First one must create a flat network, which will allow OpenStack Neutron to use actual network address space on a public or private network in a datacenter or test environment
1. Once a flat network is created, one must assign a list of non-DHCP assigned addresses that can be used for `floating-ip` addresses.
1. One must then create a router to route public/private datacenter traffic with the internet as well as OpenStack non-routable private IP address space for OpenStack tenant networks.
1. Lastly, be sure to configure the OpenStack `security-groups` to allow access to instances from the assigned `floating-ip` address.

Below are steps to create networks as described above:

```
neutron net-create --provider:physical_network=flat --provider:network_type=flat --router:external=true --shared ext-net

LOW="3";HIGH="254";ROUTER="1";NETWORK="192.168.1";CIDR=".0/24";DNS="8.8.8.8"

neutron subnet-create --name ext-net --allocation-pool start=${NETWORK}.${LOW},end=${NETWORK}.${HIGH}  --dns-nameserver ${DNS} --gateway ${NETWORK}.${ROUTER} ext-net ${NETWORK}${CIDR} --enable_dhcp=False

neutron router-create public_router

neutron router-gateway-set public_router ext-net

neutron net-create selfservice

neutron subnet-create --name selfservice --dns-nameserver ${DNS} --gateway 172.16.1.1 selfservice 172.16.1.0/24

neutron router-interface-add public_router selfservice

nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0

nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
```

## Hardening your cloud

### OpenStack Ports

<http://docs.openstack.org/liberty/config-reference/content/firewalls-default-ports.html>