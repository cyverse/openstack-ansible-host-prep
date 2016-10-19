# OpenStack Ansible Host Prep

The [OpenStack-Ansible](http://docs.openstack.org/developer/openstack-ansible/) project has [specific requirements](http://docs.openstack.org/developer/openstack-ansible/newton/install-guide/overview-requirements.html) for host layout and networking. This project, OSA Host Prep, automates most of this required configuration, prior to running OpenStack Ansible.

This is confirmed working for OpenStack Newton on Ubuntu 16.04. Your mileage may vary if you try deploying other versions/distros.

## Issues/todo/questions
- APT mirror role is broken for Ubuntu 16.04 but not needed at all (just makes the deployment faster). Either fix APT mirror role or rip it out as it's not strictly needed.
- Why do we "copy master's private key to all hosts" in the host-credentials role? This seems to violate the security guideline of not spreading SSH private keys to other hosts.
- We should not disable host key checking. learn host keys from MAAS
- Perhaps automate the process of manually connecting to each host, accepting the initial host key (checking against host key from MAAS installation), and copying deployer's public key to /root/.ssh/authorized_keys
- Document how to populate TARGET_HOSTS dictionary of group_vars/all
- I think APT mirror role is overloaded, there should be a separate role/task to deploy sources.list to the target hosts, then APT mirror role can be Galaxy-ized
- Give the host groups a consistent naming convention
- Automate some manual steps: the network connectivity testing, the partitioning of the LVM volume on the cinder node, and the disabling of the firewall
- Why does OpenStack recommend a hardware load balancer for production? The HAProxy+keepalived arrangement seems fine.

## Deployment requirements

This Ansible code will handle most of OSA's requirements, but you should still familiarize yourself with the host layout and networking requirements.

Overview: <http://docs.openstack.org/developer/openstack-ansible/newton/install-guide/overview.html>
Installation requirements: <http://docs.openstack.org/developer/openstack-ansible/newton/install-guide/overview-requirements.html>

### Host Layout

This repository leverages the OSA host layout exactly, execpt for the following differences:

* The `Deployment Host` requires identical host networking as all OSA nodes, so instead of using a separate machine, we use one of the `Infrastructure Control Plane Hosts`, i.e. `infra1`.
* Instead of a hardware load balancer, we use HAProxy and Keepalived on the `Infrastructure Control Plane Hosts` which resides on the host-level operating system of those hosts
* OSA does not deploy `Elasticsearch + Kibana`, and neither does this project. This is not required, but if desired, it must be implemented separately.
* We DO use the `Block Storage Host`, so account for that.

![Host-Layout](docs/images/environment-overview.png)

### Host Networking

Host networking: <http://docs.openstack.org/developer/openstack-ansible/newton/install-guide/overview-network-arch.html>

Configuring the switching fabric between hosts is up to you, but is straightforward. Suggestions:
- Three VLANs for the networks required for OSA (container management, tunneling, and storage)
- Another (perhaps untagged) VLAN for the host management network. The HAProxy external IP address, which provides access to Horizon Dashboard and API endpoints, can also use this subnet.
- Identical switchport configurations for each host

## Deployment Guide

### Prepare for OpenStack Deploy

1. Get Ansible on your deployment host
  ```
  $ sudo su
  $ apt-get install software-properties-common
  $ apt-add-repository ppa:ansible/ansible
  $ apt-get update
  $ apt-get install ansible
  ```

  Ensure that you can SSH to all of the target hosts using SSH key authentication, and that you have accepted their host keys into your known_hosts file. In other words, generate an SSH keypair on your deployment host, copy it to each of the target hosts' authorized_keys files, and test passwordless SSH connection from the deployment host to each target hosts.

1. Clone this repo to your deployment host, and populate the Ansible inventory file (`ansible/hosts`) with the actual hostnames and IP addresses of your target hosts. If you want to use a separate inventory file that is stored elsewhere, change line `17` of the `ansible/ansible.cfg` file to point to that host file, e.g.:

	```
	hostfile       = <your-private-repo-here>/ansible/inventory/hosts
	```

1. Prepare host networking by determining interfaces and IP addresses, and populating the `ansible/inventory/group_vars/all` with IPs and other variables. If you are storing your hosts file somewhere else, consider moving the group_vars folder, storing it alongside your hosts file.

  You can use the following commands to retrieve the network interfaces and IP addresses of your target hosts, for reference:

	```
	ansible target-hosts -m shell -a "ip addr show" > all-interfaces.txt

	cat all-interfaces.txt | grep -v "$(cat all-interfaces.txt | grep 'lo:' -A 3)"
	```

1. Set OSA_VERSION in group_vars/all to the appropriate branch or tag of the Openstack-Ansible project, or leave the default.


1. ~~Create `apt-mirror` by editing the `mirror` host group in the `ansible/inventory/hosts` in this directory and running the playbook below.~~ This role currently deploys a broken sources.list to target hosts running Ubuntu 16.04. Skip this step until the role is fixed. An APT mirror is not strictly required.

	```
	ansible-playbook playbooks/apt-mirror.yml
	```

1. Create and set credentials for all nodes for cases where manual console login is required to fix networking. (If there is a problem with the networking configuration, you need a "backdoor" into systems.)

	```
	ansible-playbook playbooks/host_credentials.yml
	```

1. Set up host networking for VLAN tagged interfaces and Linux Bridges.
	1. One might consider running the below command with the CLI argument: `--skip-tags restart-networking` and manually checking hosts to ensure proper configuration, then running `ansible target-hosts -m shell -a "ifdown -a && ifup -a"` to bounce the interfaces.

	```
	ansible-playbook playbooks/configure_networking.yml
	```

1. Test basic connectivity after network configuration
	1. Basic Tests from the deployment host

		```
		ansible target-hosts -m ping

		ansible target-hosts -m shell -a "ip a | grep -v 'lo:' -A 3"

		ansible target-hosts -m shell -a "ifconfig | grep br-mgmt -A 1 | grep inet"
		```

	1. Further manual testing (Login to a node to test bridges)

		```
		# Where X = low range and Y = high range.
		X=<low-last-octet-ip>;Y=<high-last-octet-ip>;nmap -sP 172.29.236.${X}-${Y}
		X=<low-last-octet-ip>;Y=<high-last-octet-ip>;nmap -sP 172.29.240.${X}-${Y}
		X=<low-last-octet-ip>;Y=<high-last-octet-ip>;nmap -sP 172.29.244.${X}-${Y}

    # For the following, remove/add `172.29.${subnet}.Z` as needed if your IP range is non-contiguous

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

1. To ensure a easy install, be sure to disable `ufw` or any other firewall like `iptables` on all OpenStack nodes **BEFORE** deploying OSA, as it could cause the install to hang, or fail.

	```
	ansible target-hosts -m shell -a "ufw disable"
	```

### Deploy OpenStack using OpenStack-Ansible

1. If SSH on the hosts are configured with a port other than port `22`, this `~/.ssh/config` must be used on the deployment host.  Replace all fields containining `< >` and `<SSH-PORT>` sections

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

1. Login to deployment node, and start filling out the configuration file

	```
	cd /etc/openstack_deploy/
	cp openstack_user_config.yml.example openstack_user_config.yml
	vim openstack_user_config.yml
	```

1. Follow documentation to populate configuration files here: <http://docs.openstack.org/developer/openstack-ansible/newton/install-guide/configure.html>
1. Begin filling out configuration file with `br-mgmt` IPs for each host to be used.  **DO NOT** use the host's physical IP address.
1. Fill out `openstack_user_config.yml` and `user_variables.yml`
1. Generate OpenStack Credentials found here: <http://docs.openstack.org/developer/openstack-ansible/newton/install-guide/configure.html>

	```
	cd /opt/openstack-ansible/scripts
	python pw-token-gen.py --file /etc/openstack_deploy/user_secrets.yml
	```
1. Configure HAProxy found here: <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/configure-haproxy.html#making-haproxy-highly-available>

  From here, this guide more-or-less follows the [OSA installation docs](http://docs.openstack.org/developer/openstack-ansible/newton/install-guide/installation.html). We probably shoudln't maintain parallel documentation.

1. Check syntax of configuration files: <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/configure-configurationintegrity.html>

	```
	cd /opt/openstack-ansible/playbooks/

	openstack-ansible setup-infrastructure.yml --syntax-check --ask-vault-pass
	```

1. Hosts file

1. Run Foundation Playbook: <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/install-foundation.html#running-the-foundation-playbook>

	```
	openstack-ansible setup-hosts.yml --ask-vault-pass
	```

1. Run infrastructure playbook found here: <http://docs.openstack.org/developer/openstack-ansible/install-guide/install-infrastructure.html#running-the-infrastructure-playbook>

	```
	openstack-ansible setup-infrastructure.yml --ask-vault-pass
	```

1. Manually verify that the infrastructure was set up correctly (Mainly a verification of Galera): <http://docs.openstack.org/developer/openstack-ansible/install-guide/install-infrastructure.html#verify-the-database-cluster>

	```
	ansible galera_container -m shell -a "mysql \
-h localhost -e 'show status like \"%wsrep_cluster_%\";'"

	OR

	lxc-ls | grep galera

	lxc-attach -n infra1_galera_container-XXXXXXX

	mysql -u root -p

	show status like 'wsrep_cluster%';

	# ^^ That command should display a numeric cluster size equal to the amount of infra-nodes used.
	```

1. Do not proceed if the `galera` cluster size is not equal to the amount of infra-nodes used, as it could cause deployment issues.  Be sure to resolve before proceeding to the next step.
1. Run the `playbook` to setup OpenStack found here: <http://docs.openstack.org/developer/openstack-ansible/install-guide/install-openstack.html#running-the-openstack-playbook>

	```
	openstack-ansible setup-openstack.yml --ask-vault-pass
	```

## What now?
Now that you have a running cloud, other things need to be set up in order to use OpenStack.

For steps on how to do this, see [post-deployment](docs/post-deployment.md).

## Troubleshooting Tips
### Dynamic Groups
OSA uses dynamically created groups of hosts and containers for targeting.
To see a list of groups, run the following from the deployment host:
```
source /opt/ansible-runtime/bin/activate
/opt/openstack-ansible/scripts/inventory-manage.py -G
```

### Viewing Logs
#### Viewing Ansible run logs
Check `/openstack/log/ansible-logging` on the deployment host. :)

#### Viewing logs from services provisioned by OSA
`lxc-attach` to the rsyslog container on your logging server, and look in /var/log/log-storage. Everything that logs to rsyslog, including most of the services that OSA sets up, will end up here.


### LXC Container commands

<http://docs.openstack.org/developer/openstack-ansible/newton/developer-docs/ops-lxc-commands.html>

## Deploying OpenStack Liberty -- everything below should be either worked into above sections or deprecated

* Overview: <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/overview-osa.html>
* Host layout: <http://docs.openstack.org/developer/openstack-ansible/liberty/install-guide/overview-hostlayout.html>

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
