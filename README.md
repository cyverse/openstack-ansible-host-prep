# CyVerse OpenStack Ansible Deployment

## Resources

* <http://docs.openstack.org/developer/openstack-ansible/install-guide/overview-workflow.html>
* <http://docs.openstack.org/developer/openstack-ansible/liberty/index.html>
* <https://github.com/openstack/openstack-ansible>
* <https://cunninghamshane.com/openstack-in-containers-install-and-upgrade/>

## Deploying OpenStack Liberty

* Overview: <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/overview-osa.html>
* Host layout: <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/overview-hostlayout.html>

## LXC Container commands

<http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/overview-lxc.html>

List containers and summary information such as operational state and network configuration:

```
# lxc-ls --fancy

OR

# lxc-ls -f
```

Show container details including operational state, resource utilization, and veth pairs:

```
# lxc-info --name container_name
```

Start a container:

```
# lxc-start --name container_name
```

Attach to a container:

```
# lxc-attach --name container_name
```

Stop a container:

```
# lxc-stop --name container_name
```

## Hosts

5 minimum required nodes + 1 node for cinder

```
3 - Control Plane
1 - Logging
1 - Compute
1 - Cinder
```

**Requires a deployment host (can be the primary "Infrastructure Control Plane Host", or other host with access to all VLAN interfaces)**

**Requires an HAProxy container on all Control Plane Hosts**

## Networking
Networking diagram and description found here: <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/overview-hostnetworking.html>

### Required VLAN Tags

In order to deploy OpenStack using OSA, 4 total VLAN Tags are required.

Role | Description | Number
--- | --- | ---
Native | Native tag used within your subnet | 100
Container | Tag for container management network | 101
Tunnel | Tag for tunneling network | 102
Storage (Optional) | Tag for `cinder` storage network | 103


### Target Host
Required Network bridges

* **lxcbr0** - NAT for LXC (Created Automatically)
* **br-mgmt** - Target host management (connected to bonded interface `bond0`/`Primary eth adapter`)
* **br-storage** - Segregated traffic for compute and block storage hosts (connected to bonded interface `bond0`/`Primary eth adapter`)
* **br-vxlan** - VXLAN tunnel/overlay networks (connected to bonded interface `bond1`/`Secondary eth adapter`)
* **br-vlan** - VLAN network traffic that does **NOT** have an assigned IP (connected to bonded interface `bond1`/`Secondary eth adapter`)


### Controller Container Networking Layout

This configuration may be on only a subset of containers, where as some of them will only have a single interface

* eth0 == lxcbr0
* eth1 == br-mgmt
* eth2 == br-storage
* eth10 == br-vxlan
* eth11 == br-vlan

### Neutron Controller Container

DHCP agent + L3 Agent and Linux Bridge

* eth2 == br-vxlan
* eth3 == br-vlan

## Installation Requirements

### Target hosts

* Ubuntu 14
* SSH + SSH Keys
* NTP
* Python 2.7
* Kernel > 3.13

### Cinder Target Host (Linus)

* LVM
	* volume group: `cinder-volumes` and `lxc`
	* Requires 5GB per container

## Security

* Overview: <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/overview-security.html>
* AppArmor Ansible role: <https://github.com/openstack/openstack-ansible/blob/liberty/playbooks/roles/lxc_hosts/templates/lxc-openstack.apparmor.j2>
* SSL (Using our certs): <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/configure-sslcertificates.html>
* Hardening: <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/configure-initial.html#security-hardening>

## Networking Reference Architecture

* <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/targethosts-networkrefarch.html>
* <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/targethosts-networkexample.html>

Interface | IP | Bridge Interface | Manual?
--- | --- | --- | ---
eth{primary} | 10.1.1.10 | N/A | Yes
lxcbr0 | Unnumbered | eth{primary} | No
br-mgmt | 192.168.100.10 | eth{primary}.101 | Yes
br-vxlan | 192.168.95.10 | eth{secondary}.102 | Yes
br-storage | 192.168.85.10 | eth{primary}.103 | Yes
br-vlan | Unnumbered | eth{secondary} | Yes

### Physical Interfaces Files

* /etc/network/interfaces
	
	```
	auto lo
	iface lo inet loopback
		
	source /etc/network/interfaces.d/*
	```

* /etc/network/interfaces.d/device-eth

	```
	# Primary Interface / Bond
	auto eth{primary}
	iface eth{primary} inet static
	  address 10.1.1.10
	  description management interface
	  broadcast 10.1.1.255
	  gateway 10.1.1.1
	  netmask 255.255.255.0
	  network 10.1.1.0
	  dns-nameservers 8.8.8.8
	  dns-search domain.com

	# Container Management VLAN
	auto eth{primary}.101
	iface eth{primary}.101 inet manual
	  vlan-raw-device eth{primary}

	# Container Storage VLAN
	auto eth{primary}.103
	iface eth{primary}.103 inet manual
	  vlan-raw-device eth{primary}	  

	# Secondary Interface / Bond
	auto eth{secondary}
	iface eth{secondary} inet manual
	  up ip link set dev $IFACE up
	  down ip link set dev $IFACE down

	# Container Tunnel VLAN
	auto eth{primary}.102
	iface eth{primary}.102 inet manual
	  vlan-raw-device eth{secondary}
	```
	
* /etc/network/interfaces.d/device-bridges

	```
	# Container management bridge
	auto br-mgmt
	iface br-mgmt inet static
		bridge_stp off
		bridge_waitport 0
		bridge_fd 0
		# Bridge port references tagged interface
		bridge_ports eth{primary}.101
		address 192.168.100.10
		netmask 255.255.255.0
		dns-nameservers 8.8.8.8
	
	# OpenStack Networking VXLAN (tunnel/overlay) bridge
	auto br-vxlan
	iface br-vxlan inet static
		bridge_stp off
		bridge_waitport 0
		bridge_fd 0
		# Bridge port references tagged interface
		bridge_ports eth{primary}.102
		address 192.168.95.10
		netmask 255.255.255.0
	
	# OpenStack Networking VLAN bridge
	auto br-vlan
		iface br-vlan inet manual
		bridge_stp off
		bridge_waitport 0
		bridge_fd 0
		# Bridge port references untagged interface
		bridge_ports eth{secondary}
	
	# Storage bridge (optional)
	auto br-storage
	iface br-storage inet static
		bridge_stp off
		bridge_waitport 0
		bridge_fd 0
		# Bridge port reference tagged interface
		bridge_ports eth{primary}.103
		address 192.168.85.10
		netmask 255.255.255.0
	```

### IP Layout

Hostname | Interface | IP | Bridge
--- | --- | --- | ---
`external_lb_vip` | N/A | 10.1.1.2 | eth{primary} network
`internal_lb_vip` | N/A | 192.168.100.10 | br-mgmt IP
infra_control_plane_host | eth{primary} | 10.1.1.10 | N/A
 | br-mgmt | 192.168.100.10 | eth{primary}.101
 | br-storage | 192.168.95.10 | eth{primary}.103
 | br-vxlan | 192.168.85.20 | eth{secondary}.102
 | br-vlan | Unnumbered | eth{secondary}
 | eth{secondary} | Unnumbered | N/A
 | infra1_container | 192.168.100.10 | br-mgmt
 
## Configure Targets

1. Install package dependencies
	1. In this repo, use `configure_targets.yml` playbook
		
		```
		ansible-playbook configure_targets.yml -e "SSH_PUBLIC_KEY='ssh-rsa AAAA...'"
		```
		
1. Set up NTP

## Set up Deployment Host

1. Find the latest stable TAG: <https://github.com/openstack/openstack-ansible/releases> and verify that the selected tag corresponds with the version of OS one wishes to deploy.  One may see something similar to this: `meta:series: liberty` in the release notes.
1. Clone repo on deploy host (This can be done via Ansible, or on one of the "Infrastructure Control Plane Host")
	
	```
	git clone -b 12.0.9 https://github.com/openstack/openstack-ansible.git /opt/openstack-ansible
	```
	
1. Run bootstrap

	```
	cd /opt/openstack-ansible
	scripts/bootstrap-ansible.sh
	```	

## Prepare Target Hosts

Re-run Ansible Playbook to include changes for block-storage node

1. Run Ansible Playbook to set up Bare-Metal host credentials, SSH keys and root passwords.  (Very important to configure root password, so that recovery of configuration is still possible from the host's console)

	```
	cd ansible
	ansible-playbook playbooks/host_credentials.yml
	```
	
1. Configure Bare-Metal host networking for OSA setup (VLAN tagged interfaces and LinuxBridges).  At this point, you MUST modify the `hosts` file AND `group_vars/all` variables under `TARGET_HOST_NETWORKING`, `TARGET_HOSTS` and `CINDER_PHYSICAL_VOLUME` sections.

	```
	ansible-playbook playbooks/configure_networking.yml
	```
	
1. Prepare hosts for OSA Deployment.  This Playbook configures the Deployment host AND OSA Target-hosts.  (Ensure that `hosts` and `group_vars/all` are filled out and accurate)

	```
	ansible-playbook playbooks/configure_targets.yml
	```