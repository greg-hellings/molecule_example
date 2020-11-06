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

Walkthrough
-----------

### Create A Role

To start from the base role, checkout the role at that level:

```bash
git checkout bare_role
# Be sure that ansible and molecule are installed
pip install molecule ansible
```

### Create default scenario

Next, we create a molecule scenario

```bash
# Be sure that the driver is installed
pip install molecule-podman
molecule init scenario --driver-name podman
# Populate test files as appropriate
```

Change the `verify.yml` to check that the user directory has been
created properly.

```yaml
- name: Verify
  hosts: all
  gather_facts: false
  tasks:
    - name: Run stat
      stat:
        path: /home/test_user
      register: _is_test_user

    - name: Be sure it exists
      assert:
        that: _is_test_user.stat.exists
```

Now run the default scenario with `molecule test`

The repo can be seen at that stage by doing a `git checkout first_scenario`

### Create Scenario With New Name

Now create a new scenario, with a new name, to test that the name of
the user created is correct.

```bash
molecule init scenario --driver-name podman configure_name
```

Add a var to the `molecule/configure_name/converge.yml` to set the username
to something other than the default. And add the same verify.yml code above,
but adjust it to look for the proper pathname to match the name you've
configured.

Now run the scenario by specifying the name

```bash
molecule test -s configure_name
```

This can also be found, now, with `git checkout second_scenario`

### Consolidate molecule.yml into shared file

Note that there is significant overlap in the molecule.yml files of these
two scenarios. With molecule we can specify a root configuration file. The
tool will do a deep merge with the molecule.yml file that's in the scenario
folder, so anything can be overridden at even the lowest level.

Currently the two files are identical, so we can just copy one of the
molecule.yml files to the root and now we can completely empty out the files
in each scenario. It's best to keep some minimal information in the file,
but it is not necessary.

Let's improve things by forcing Ansible to output color. We do this by
setting `ansible.cfg` values in the provisioner section. We do it in this
manner:
```yaml
provisioner:
  config_options:
    defaults:
      force_color: true
```
Any config options that would go into `ansible.cfg` can live here. Some of
my favorites are `stdout_callback: yaml` and `force_color: true` and
`remote_tmp: /tmp/${USER}/ansible`.

Now we can test by running the two scenarios:

```bash
molecule -c molecule.yml test
molecule -c molecule.yml test -s configure_name
```

This can also be found with `git checkout consolidated_molecule`

### Combine converge.yml and verify.yml

So, in an effort to reduce duplication, let's use a single converge.yml and
verify.yml file. How can we do this? By using `group_vars` in the
molecule.yml.

Move `molecule/default/converge.yml` into `molecule/shared/converge.yml`. Do
the same with `molecule/default/verify.yml` and let's get rid of the
`molecule/configure_name/{converge,verify}.yml`.

Now, tell Molecule where to find the converge and verify files in the
root molecule.yml by modifying the `provisioner` section in this manner.

```yaml
provisioner:
  playbooks:
    converge: ../shared/converge.yml
    verify:  ../shared/verify.yml
```

Now, go into `molecule/configure_name/molecule.yml` and let's take advantage
of the deep merging that Molecule does. We can add this code to that file,
and it will set the `group_vars` appropriately. The relevant file should now
look like this

```yaml
provisioner:
  inventory:
    group_vars:
      all:
        molecule_example_user: greg
```

Now you should notice that the default scenario runs properly, but the
configure\_name scenario is failing. This is because `verify.yml` is hard
coded to look for a particular path. To parameterize this we can add another
variable telling the path to look for. This time, we can add it to both
files in the same section as above. Add a line with the name "verify\_path"
and the value "/home/greg" to the `molecule/configure_name/molecule.yml`
file. Add the whole above section, minus the `molecule_example_user` line
to `molecule/default/molecule.yml` and set the value of `verify_path` to
`/home/test_user`.

This portion can be seen as `git checkout deduplicate`.

### Other Advanced Uses

There are many other advanced uses you can configure with Molecule. You can
set dependencies on other roles or collections in Ansible Galaxy. These are
configured in a `requirements.yml` that lives in the root of the Molecule
scenario. Its location can be changed by setting these values:

```yaml
dependency:
  options:
    role-file: path/relative/to/role_dir  # For roles
    requirements-file: path/relative/to/role_dir  # For collections.
```

Of course, those two files can be combined into one if you use the proper
syntax inside of it like

```yaml
roles:
  - some.role
collections:
  - some.collection
```

It is possible to write custom create and destroy playbooks. These can be
configured in the `provisioner.playbooks` section with names like one might
expect (`create`, `destroy`, etc). You can even have different provision
scripts for each driver type. That structure would look something like this.
Paths are relative to the scenario's directory.

```yaml
provisioner:
  playbooks:
    podman:
      create: podman-create.yml
      destroy: podman-destroy.yml
    docker:
      create: ../path/to/docker-create.yml
      destroy: ../path/to/docker-destroy.yml
```

In addition to Ansible verifiers, you can also write verifiers in Python
using the testinfra package. This is the old default for Molecule 2 code,
but the default has changed to using Ansible playbooks for the verify stage.

Writing linters is possible. Currently lint is just a string that is passed
as a shell script to the underlying system shell. You specify it in a
`molecule.yml` file as such:

```yaml
lint: |-
  set -ex
  ansible-lint .
  yamllint .
  # etc, commands executed from role directory
```
