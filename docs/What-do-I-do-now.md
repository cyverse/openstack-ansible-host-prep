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

## Hardening your cloud

### OpenStack Ports

<http://docs.openstack.org/liberty/config-reference/content/firewalls-default-ports.html>