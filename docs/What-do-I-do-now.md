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

For more information on how VRRP works, see these documents:

* <http://docs.openstack.org/liberty/networking-guide/scenario-l3ha-lb.html>
* <https://wiki.openstack.org/wiki/Neutron/L3_High_Availability_VRRP>

For steps on how to troubleshoot OpenStack Neutron Networking, see these pages:

* <http://docs.openstack.org/ops-guide/ops_network_troubleshooting.html>

### Launch an instance with Networking

#### From Horizon

1. Login to OpenStack `Horizon` Web-interface defined in the variable `external_lb_vip_address` in the `openstack_user_config.yml` file under the section called `global_overrides`
1. Grab the login password from the `user_secrets.yml` file listed under `keystone_auth_admin_password`.  Login using `admin` and the `keystone_auth_admin_password `.
1. Select `Project`, `Compute/Instances`
1. Select `Launch Instance`
1. Fill out the following settings:

	```
	# Details
	Availability Zone: nova
	Instance Name: <your-instance-name-here>
	Flavor: <m1.medium for non-cirros images, m1.small for cirros>
	Instance Count: 1
	Instance Boot Source: Boot from image
	Image Name: <glance image to boot>
	
	# Access & Security
	Click + to add your id_rsa.pub key
	Security Groups: Select default
	
	# Networking
	Drag "selfservice" network into "Selected networks"
	
	Launch
	
	# Add Floating IP address
	Click down-arrow under "Actions" for that launched instance, and select "Associate Floating IP"
	Click "+" to create a Floating IP allocation
	Pool: ext-net

	Allocate IP
	
	Associate
	```

#### From OpenStack CLI

1. <http://docs.openstack.org/liberty/install-guide-ubuntu/launch-instance.html>
1. <http://docs.openstack.org/liberty/install-guide-ubuntu/launch-instance-private.html>

### Verify Operabilty

1. Verify that instance shows an "Active" status, if not, check all OpenStack Logs by logging into the `Infrastructure Logging Host` and check for errors, like so:

	```
	cd /openstack/log1_rsyslog_container-<container-id>/log-storage

	tail -n 1000 -f */*.log | grep ERROR
	```

## Adding Compute Nodes
Once one has verfied that OpenStack is working well with one node, one might need to add additional compute nodes to handle the load from a large amount of users.

In order to do this, one will need to re-run all steps defined on the main `README.md` page while `--limit "<compute-bare-metal-2>,<compute-bare-metal-3>,<compute-bare-metal-4>"` at every step.

Once finished, update the `openstack_user_config.yml` to include the new `used_ips` for those hosts, and add them individually to the `compute_hosts`.

Then run:

```
openstack-ansible setup-everything.yml --limit "<compute2>,<compute3>,<compute4>"
```

For more information, see how to do this here: <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/ops-addcomputehost.html>

## Quick Troubleshooting Tips

1. If `Horizon` reports a failure on instance launch because it cannot select a Hypervisor (or says there is not available), check the logs on the for the following message: 

	```
	Image <glance-image-id> could not be found.
	```
	
	This means that HAProxy sent a request to a `Glance` container which did not have the image requested.  Until something like `glance-irods` is installed and configured, one might want to disable HAProxy for `Glance` to only select the node that has all of the images.

	To fix, this do the following:
	
	Login to all `Infrastructure Control Plane Hosts` at once using broadcast input with your favorite terminal, or `tmux` with `setw syncronize-panes on` modify the HAProxy configuration.
	
	```
	cd /etc/haproxy/conf.d/
	
	lxc-attach -n infra<LITERAL-TAB>_glance_container-<LITERAL-TAB>
	
	cd /var/lib/glance/images/
	
	ls -la
	
	# Identify the container that contains all the Glance images in it, and take note of this container
	
	exit
	
	vim glance_registry
	
	# Modify the section labeled: "backend glance_registry-back" and comment out the two containers that DO NOT have the correct Glance images on them.
	
	# E.g. Where Glance "infra3" contained all Glance images
	backend glance_registry-back
	    mode http
	    balance leastconn
	    #server infra1_glance_container-<id> 172.29.239.<ip1>:9191 check port 9191 inter 12000 rise 3 fall 3
	    #server infra2_glance_container-<id> 172.29.238.<ip2>:9191 check port 9191 inter 12000 rise 3 fall 3
	    server infra3_glance_container-<id> 172.29.239.<ip3>:9191 check port 9191 inter 12000 rise 3 fall 3
	
	```

1. If all of the `Infrastructure Control Plane Hosts` go down or reboot at once, this will have disasterous affect on Galera MySQL Cluster, as it will likely not recover, and have split-brain issues.  To fix this, follow the steps listed here: <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/ops-galera-recoverysingle.html>


## Hardening your cloud

### OpenStack Ports

<http://docs.openstack.org/liberty/config-reference/content/firewalls-default-ports.html>

At this point, all firewalls will be down, so one will need to be sure to configure `ufw` or `iptables` on all the hosts.

## Other Tools

### Other OpenStack Releases

<http://docs.openstack.org/developer/openstack-ansible/>

### IRC Help Forums

<https://wiki.openstack.org/wiki/IRC>

1. To get help, go here: <http://webchat.freenode.net/?channels=openstack-ansible>
1. Pick a username and chat! 

### RabbitMQ Management Console

<https://www.rabbitmq.com/management.html>