molecule_example
===========

A sample role that we will use to create Molecule scenarios and train
people on how to properly use Molecule

Requirements
------------

Ansible 2.8 or higher

Role Variables
--------------

Currently the following variables are supported:

### General

* `molecule_example_user` - Default: test\_user. Name of the user and group
  to create
* `molecule_example_become` - Default: true. If this role needs administrator
  privileges, then use the Ansible become functionality (based off sudo).
* `molecule_example_become_user` - Default: root. If the role uses the become
  functionality for privilege escalation, then this is the name of the target
  user to change to.

Dependencies
------------

None

Example Playbook
----------------

```yaml
- hosts: molecule_example-servers
  roles:
    - role: molecule_example
```

License
-------

GPLv3

Author Information
------------------

Greg Hellings <greg.hellings@gmail.com>
