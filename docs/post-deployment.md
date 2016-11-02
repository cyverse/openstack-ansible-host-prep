# Post-Deployment Steps
Now that you have a (hopefully) functional OpenStack Cloud, you need to do a couple of things to get your cloud fully operational and usable for applications like Atmosphere.

## Back up your database
In the event that all of your galera containers are shut down at the same time, the galera cluster will break and you may need to restore from a backup. Take an initial dump of your database in case this happens. From inside a galera container:
`mysqldump --opt --events --all-databases > openstack-YYYYMMDD.sql`, and store it in a safe place.

## Infrastructure Requirements

### IGMP snooping querier
If using Cisco gear with this deployment setup, you must configure an IGMP snooping querier on the same subnet as the tunneling subnet and VLAN.

For example, if using the default range of IP addresses defined in the OpenStack Ansible deployment, i.e. `tunnel: 172.29.240.0/22`, configure an IGMP snooping querier within that range on the VLAN used for the tunnelling network, or `Multicast` traffic for `HA L3 Neutron agents` will not work. It is suggested to set your `IGMP Snooping Querier` IP to `172.29.243.254` (if using the above tunnel block of IP addresses).

### Example Dell (Force10) switch configuration
```
interface Vlan 102
 name tunnel
 ip address 172.29.243.254/22
 ip igmp snooping querier
 no shutdown
!
ip igmp snooping enable
```

This snippet does not cover adding tagged/untagged trunk ports to the VLAN interface, which you must do specific to your deployment.

## Setting up OpenStack for general use

### Interacting with OpenStack CLI

SSH to any of the infrastructure hosts. Then, attach one of the utility containers and source the openrc file:

```
lxc-attach -n name-of-your-utility-container
source /root/openrc
```

Then, you can use the OpenStack command-line interface -- here's a [cheat cheet](http://docs.openstack.org/user-guide/cli-cheat-sheet.html).

### Creating OpenStack Glance Images

See [Verify operation of Glance](http://docs.openstack.org/newton/install-guide-ubuntu/glance-verify.html).

For the lazy, here are commands to set up Ubuntu 16.04 and CentOS 7 images:

```
wget https://cloud-images.ubuntu.com/xenial/current/xenial-server-cloudimg-amd64-disk1.img
openstack image create "ubuntu-16.04" --file xenial-server-cloudimg-amd64-disk1.img --disk-format qcow2 --container-format bare --public
wget http://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud-1608.qcow2
openstack image create "centos-7" --file CentOS-7-x86_64-GenericCloud-1608.qcow2 --disk-format qcow2 --container-format bare --public
```

OpenStack should now report all of the images you just created:

```
root@infra1_utility_container-<ID>:~# openstack image list
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

# Fill in these with your external (public) IP space
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

* <http://docs.openstack.org/newton/networking-guide/scenario-l3ha-lb.html>
* <https://wiki.openstack.org/wiki/Neutron/L3_High_Availability_VRRP>

For steps on how to troubleshoot OpenStack Neutron Networking, see these pages:

* <http://docs.openstack.org/ops-guide/ops_network_troubleshooting.html>

### Launch an instance with Networking

#### From Horizon

1. Login to OpenStack `Horizon` Web-interface defined in the variable `external_lb_vip_address` in the `openstack_user_config.yml` file under the section called `global_overrides`
1. Grab the login password from the `user_secrets.yml` file listed under `keystone_auth_admin_password`.  Login using `admin` and the `keystone_auth_admin_password `.
1. Create a flavor, System -> Flavors
1. Select `Project`, `Compute/Instances`
1. Select `Launch Instance`
1. Fill out the following settings:

	```
	# Details
	Availability Zone: nova
	Instance Name: <your-instance-name-here>
	Flavor: <use the flavor you created, m1.medium for non-cirros images, m1.small for cirros>
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

1. <http://docs.openstack.org/newton/install-guide-ubuntu/launch-instance.html>

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

For more information, see how to do this here: <http://docs.openstack.org/developer/openstack-ansible/newton/developer-docs/ops-add-computehost.html>

## Troubleshooting

### Troubleshooting Instance Launching

1. If `Horizon` reports a failure on instance launch because it cannot select a Hypervisor (or says there is not available), check the logs on the for the following message:

	```
	Image <glance-image-id> could not be found.
	```

	This means that HAProxy sent a request to a `Glance` container which did not have the image requested.



  Until something like `glance-irods` is installed and configured, one might want to disable HAProxy for `Glance` to only select the node that has all of the images.

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

### Galera cluster loses quorum
If all of the `Infrastructure Control Plane Hosts` go down or reboot at once, this will have disastrous affect on Galera MySQL Cluster, as it will likely not recover, and have split-brain issues. See [Galera cluster recovery](http://docs.openstack.org/developer/openstack-ansible/newton/developer-docs/ops-galera-recovery.html) for more information.

## Hardening your cloud

### OpenStack Ports

<http://docs.openstack.org/liberty/config-reference/content/firewalls-default-ports.html>

At this point, all firewalls will be down, so one will need to be sure to configure `ufw` or `iptables` on all the hosts.

## Other Tools

### Other OpenStack Releases

<http://docs.openstack.org/developer/openstack-ansible/>

### IRC Help Forums

<https://wiki.openstack.org/wiki/IRC>

### RabbitMQ Management Console

<https://www.rabbitmq.com/management.html>
