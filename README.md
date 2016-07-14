# CyVerse OpenStack Ansible Deployment

## Prepare for OpenStack Deploy

1. Create `apt-mirror` by editing the `mirror` host group in the `ansible/inventory/hosts` in this directory and running the playbook below.

	```
	ansible-playbook playbooks/apt-mirror.yml -i inventory/hosts
	```

1. Create and set credentials for all nodes for cases where manual login is required to fix networking.  **IMPORTANT!!** if there is a problem with the networking configuration, it is critical to have a "backdoor" into systems if things go wrong.

	```
	ansible-playbook playbooks/host_credentials.yml
	```

1. Prepare host networking by determining interfaces and IP addresses, and populating the `ansible/inventory/group_vars/all` with IPs and other variables

	```
	ansible target-hosts -m shell -a "ip addr show" > all-interfaces.txt
	
	cat all-interfaces.txt | grep -v "$(cat all-interfaces.txt | grep 'lo:' -A 3)"
	```

1. Set up host networking for VLAN tagged interfaces and Linux Bridges.
	1. One might consider running the below command with the CLI argument: `--skip-tags restart-networking` and manually checking hosts to ensure proper configuration.
	
	```
	ansible-playbook playbooks/configure_networking.yml # --skip-tags restart-networking
	```

1. Test basic connectivity after network configuration
	1. Basic Tests
	
		```
		ansible target-hosts -m ping
		
		ansible target-hosts -m shell -a "ip a | grep -v 'lo:' -A 3"
		
		ansible target-hosts -m shell -a "ifconfig | grep br-mgmt -A 1 | grep inet"
		
		# Where X = low range and Y = high range.
		nmap -sP 172.29.236.X-Y
		nmap -sP 172.29.240.X-Y
		nmap -sP 172.29.244.X-Y
		```
	1. Further manual testing (Login to a node to test bridges) 
 
		```
		interface="br-mgmt" ; subnet="236" ; for i in 172.29.${subnet}.{X..Y} 172.29.${subnet}.Z;do echo "Pinging host on ${interface}: $i"; ping -c 3 -I $interface $i;done
		
		interface="br-vxlan" ; subnet="240" ; for i in 172.29.${subnet}.{X..Y} 172.29.${subnet}.Z;do echo "Pinging host on ${interface}: $i"; ping -c 3 -I $interface $i;done
		
		interface="br-storage" ; subnet="244" ; for i in 172.29.${subnet}.{X..Y} 172.29.${subnet}.Z;do echo "Pinging host on ${interface}: $i"; ping -c 3 -I $interface $i;done
		```
1. Manually partition `Block-storage` node's LVM Volume for `cinder`

	```
	/sbin/parted /dev/sd<device> -s mklabel gpt
	/sbin/parted /dev/sd<device> -s mkpart primary 0% 100%
	/sbin/parted /dev/sd<device> -s set 1 lvm on
	/sbin/parted /dev/sd<device> -s p
	```

1. Configure and prep all nodes including deployment node for OSA deploy
	1. If this role is to take care of LVM creation for `cinder-volumes` be sure to enable (i.e. uncomment) the `CINDER_PHYSICAL_VOLUME.create_flag` in `ansible/inventory/group_vars/all`

	```
	ansible-playbook playbooks/configure_targets.yml
	```

## Deploy OpenStack using OpenStack Ansible Deployment

1. Login to deployment node, and start filling out the configuration file

	```
	cd /etc/openstack_deploy/
	cp openstack_user_config.yml.example openstack_user_config.yml
	vim openstack_user_config.yml
	```

1. Follow documentation and configuration files for setting up the cloud here: <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/configure-networking.html>

1. Organize the hosts for OpenStack in the order defined here: <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/overview-hostlayout.html>
	1. It is often helpful to diagram out, or use a table to assign roles for each node.
1. Begin filling out configuration file with `br-mgmt` IPs for each host to be used.  **DO NOT** use the host's physical IP address.
1. Fill out `openstack_user_config.yml` and `user_variables.yml`
1. Generate OpenStack Credentials found here: <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/configure-creds.html>

	```
	cd /opt/openstack-ansible/scripts
	python pw-token-gen.py --file /etc/openstack_deploy/user_secrets.yml
	``` 
1. Configure HAProxy found here: <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/configure-haproxy.html>
1. Check syntax of configuration files: <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/configure-configurationintegrity.html>

	```
	cd /opt/openstack-ansible/playbooks/
	
	openstack-ansible setup-infrastructure.yml --syntax-check
	```
1. If SSH on the hosts are configured with a port other than port `22`, this `~/.ssh/config` must be used.  Replace all fields containining `< >` and `<SSH-PORT>` sections

	```
	Host 172.29.236.<IP-RANGE-HERE>?
        User root
        Port <SSH-PORT>

	Host 172.29.236.<INDIVIDUAL-IP-HOST-HERE>
	        User root
	        Port <SSH-PORT>
	
	Host *
	        User root
	        Port 22
	```

1. Run Foundation Playbook: <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/install-foundation-run.html>

	```
	openstack-ansible setup-hosts.yml
	```

1. **IMPORTANT** If no hardware load balancer is used (as defined here in this documentation), one must run this `playbook` to configure HAProxy on the Infrastructure Hosts

	```
	openstack-ansible haproxy-install.yml
	```

1. Run infrastructure playbook found here: <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/install-infrastructure-run.html>

	```
	openstack-ansible setup-infrastructure.yml
	```

1. Manually verify that the infrastructure was set up correctly (Mainly a verification of Galera): <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/install-infrastructure-verify.html>

	```
	lxc-ls | grep galera
	
	lxc-attach -n infra1_galera_container-XXXXXXX
	
	mysql -u root -p
	
	show status like 'wsrep_cluster%';
	
	# ^^ That command should display a numeric cluster size equal to the amount of infra-nodes used.
	```

1. **DO NOT** proceed to this step if the `galera` cluster size is not equal to the amount of infra-nodes used, as it could cause deployment issues.  Be sure to resolve the playbooks above before proceeding to the next step.
1. Run the `playbook` to setup OpenStack found here: <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/install-openstack-run.html>

	```
	openstack-ansible setup-openstack.yml
	```

## Post-deploy things to do

1. Secure services with SSL: <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/configure-sslcertificates.html>
 
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
	
1. Run Playbook to set up an Ubuntu `apt-mirror` using a completely separate host (could use a target-host, but it is not recommended)

	```
	ansible-playbook playbooks/apt-mirror.yml --skip-tags "update"
	
	Manually execute the command below on apt-mirror host, since it could take a very long time (upwards of 4 hours)
	
	su - apt-mirror -c apt-mirror	
	```

1. Prepare hosts for OSA Deployment.  This Playbook configures the Deployment host AND OSA Target-hosts.  (Ensure that `hosts` and `group_vars/all` are filled out and accurate)

	```
	ansible-playbook playbooks/configure_targets.yml
	```
	
1. Manually copy and enable configuration file for OSA

	```
	cd /etc/openstack_deploy/ && cp openstack_user_config.yml.example openstack_user_config.yml
	```