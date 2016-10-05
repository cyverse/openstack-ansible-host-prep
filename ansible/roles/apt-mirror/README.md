APT Mirror
=========

Configures an APT mirror for Ubuntu 14.04 or 16.04. Mirrors packages for the major distro version of the host that you install the APT mirror on. Also deploys a new sources.list
to another group of target hosts.

Currently broken in that hosts which use this APT mirror won't be able to download repositories specifically marked as amd64 architecture. Will fix later.

Intended to be used as part of https://github.com/cyverse/openstack-ansible-host-prep, may not work on its own yet. Also missing some documentation -- work in progress.

[![Build Status](https://travis-ci.org/CyVerse-Ansible/ansible-apt-mirror.svg?branch=master)](https://travis-ci.org/CyVerse-Ansible/ansible-apt-mirror)
[![Ansible Galaxy](https://img.shields.io/badge/ansible--galaxy-apt--mirror-blue.svg)](https://galaxy.ansible.com/CyVerse-Ansible/ansible-apt-mirror/)


Requirements
------------

Any pre-requisites that may not be covered by Ansible itself or the role should be mentioned here. For instance, if the role uses the EC2 module, it may be a good idea to mention in this section that the boto package is required.

Role Variables
--------------

A description of the settable variables for this role should go here, including any variables that are in defaults/main.yml, vars/main.yml, and any variables that can/should be set via parameters to the role. Any variables that are read from other roles and/or the global scope (ie. hostvars, group vars, etc.) should be mentioned here as well.

| Variable                | Required | Default | Choices                   | Comments                                 |
|-------------------------|----------|---------|---------------------------|------------------------------------------|
| MIRROR_HOSTS            | yes      | N/A     | true, false               | example variable                         |
| bar                     | yes      |         | eggs, spam                | example variable                         |

Dependencies
------------

A list of other roles hosted on Galaxy should go here, plus any details in regards to parameters that may need to be set for other roles, or variables that are used from other roles.

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: all
      roles:
         - ansible-apt-mirror

License
-------

See license.md

Author Information
------------------

https://cyverse.org
