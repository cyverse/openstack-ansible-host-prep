# OpenStack Services

## Bare Metal

Nova:

```
/openstack/venvs/nova-12.0.10/bin/python /openstack/venvs/nova-12.0.10/bin/nova-compute --log-file=/var/log/nova/nova-compute.log
```

Neutron:

```
/openstack/venvs/neutron-12.0.10/bin/python /openstack/venvs/neutron-12.0.10/bin/neutron-linuxbridge-agent --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini --config-file /etc/neutron/plugins/ml2/linuxbridge_agent.ini --log-file=/var/log/neutron/neutron-linuxbridge-agent.log
```

## Containers

Find container:

```
lxc-ls -f
```