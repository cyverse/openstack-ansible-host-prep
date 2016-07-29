# What do I do now?

Now that you have a fully functional OpenStack Cloud, you need to do a couple of things to get your cloud fully operational and usable for applications like Atmosphere.

## Ensure Infrastructure Requirements

### Cisco Switch/Router Requirements

**IMPORTANT** If one is using Cisco gear with this deployment setup, it is an absolute requirement to have a IGMP Snooping Querier on the **SAME** subnet as the tunneling VLAN network.  For example, if using the default range of IP addresses defined in the OpenStack Ansible deployment, i.e. `tunnel: 172.29.240.0/22`, one **MUST** have an `IGMP Snooping Querier` within that range, or `Multicast` traffic for `HA L3 Neutron agents` will **NOT** work!

To prevent this, it is suggested that one set their `IGMP Snooping Querier` IP to `172.29.243.254`.  Just note that this address only applies if using the above tunnel block of IP addresses.