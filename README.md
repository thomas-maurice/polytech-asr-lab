# ASR lab project

The point of this lab is to apply automation techniques to deploy a [consul](https://consul.io) cluster onto your lab virtual machines. You will use [Ansible](https://ansible.com) for that.

You will then use the created Consul cluster to register services onto them, and use Consul's [service discovery mechanism](https://www.consul.io/use-cases/discover-services) to discover them.

## Context

### Anatomy of a Consul cluster
A consul cluster is comprised of 2 main components:
* A set of `master` nodes, they will ensure quorum and maintain consistency accross the cluster
* A set of agents, which are basically all your other nodes

In this lab work you will deploy a consul master or an agent onto your lab machines, and register a service onto them, that can be found using the built in service discovery mechanism. The registered service can be whatever you like, such as a simple nginx server or what have you.

### This ansible skeleton repo
You should use this ansible skeleton repository as the base for your work, it contains a bunch of boilerplate code that might come in handy to you. Each `question` you'll have to answer should probably be living in it's own `role` (more after that later). Before you start you should generate an SSH key for your group and add it into your VM's `/root/.ssh/authorized_keys` files, you will use it to connect to the VM with ansible. You do that like so
```
$ ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -C "your.name@polytech-lille.net" -P ''
# This generates a key with no passowrd for the purpose of the lab, in the real world you'll want one
```

Then copy the content of `~/.ssh/id_ed25519.pub` to your VM's `authorized_keys`

Next let's have a look at the anatomy of this repo

#### `inventory`
Contains the list of your targets, you should specify an inventory name, which should be your hostname, and eventually a bunch of arguments, such as `ansible_ssh_host="X.Y.Z.A"` to specify the address ansible has to connect to. In our usecase you will need to have only one entry in this file.

#### `{group,host}_vars`
These directory contains variables that will be available into your ansible code. We will not dive into groups today, just know the special group `all` and it's associated file `group_vars/all.yaml` contains variables that will be available to *all* your hosts.

The file you will likely care about is the one living in `host_vars/your-vm.yaml` (the name must match the name of your machine in the inventory). In this you can add yaml formatted variables, for example:
```yaml
---
hostname: foo
bar:
    baz: 42
```

To reference these variables in your roles and in your templates, you would respectively do this `{{ hostname }}` or `{{ bar.baz }}`.

*The variables in the `host_vars` file override the one form the groups in a more specific to less specific fashion*

#### `polytech.yaml`
This is the playbook you will run, essentially the list of roles you want to apply to the host.

#### `roles`
Directory that contains the roles you want to install

#### `roles/base`
Base role, set as an example. It will install a bunch of packages on the target system as well as setting the hostname of the machine properly.

A role is a set of `tasks`, which are for example "install packages on the system", "copy a file over", "create a file from a template", "apply modes to a file", "create a docker container" and so on.

Motre information about how the directory of a role is structured is available [here](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html).

But as a TL;DR:
* `tasks/main.yaml`: List of tasks to run
* `files/*`: Files available to copy over, as is
* `templates/*`: Files that will be rendered from templates on the system
* `defaults/main.yaml`: Defaults variables if your role uses them

With this you should be good to go.

### Installing ansible & applying your work
#### Installing ansible
I like to make it work inside a virtualenv, for simplicity, so create a virtualenv activate it and install ansible.
```
$ python3 -m venv ~/env-ansible
$ . ~/env-ansible/bin/activate
$ pip3 install -U setuptools wheel
$ pip3 install -U ansible
```

#### Running your playbook
```
$ ansible-playbook -v -i inventory polytech.yaml
```

## Actual lab work
### Write an SSH key role
You need to write a role, call it `roles/ssh_key` that drops a [template](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html) that contains all the SSH keys in the `group_vars/all.yaml` file into a file in `/root/.ssh/authorized_keys`. Careful, the `/root/.ssh` directory might not exist.

:warning: You wan't to put the ssh public keys in there, one per line.

You also need to ensure the file has `0600` permissions on it otherwise SSH will not accept to use it because the permissions are too open.

You will want to leave my own SSH key into the file, so I can help out with debugging.

:information_source: You might want to check out the Jinja2 templating syntax

### Install Docker using a Docker role
Using `ansible-galaxy`, install the Docker role (the one by [Jeff Geerling](https://galaxy.ansible.com/geerlingguy/docker) works well)

### Firewall your hosts
Create a role to firewall your instances from the outside Internet. You can skip this altogether if you don't have any routed IP attached to your VM. You want a minima to forbid the following traffic from reaching your host from the Internet (while keeping it accessible from the lab VLANs obviously):
* 8600/tcp
* 8600/udp
* 8500/tcp
* 8501/tcp
* 8502/tcp
* 8301/tcp
* 8301/udp
* 8302/tcp
* 8302/udp
* 8300/tcp

More infos about why [here](https://www.consul.io/docs/install/ports). You can achieve this using the [iptables](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/iptables_module.html) Ansible module.

### Determine who will be running an agent and who will be running a server
The 3 first groups reaching this point of the lab will be running servers. Not much config change is required, but we must have at least 3 servers set up. Let Xavier ot Thomas know who is doing what.

### Install Consul
#### Prepare running consul
You should create a `consul` user on your system and give it a new UID and GID, for simplicity purposes you will create the `consul` groups and users with the respective UID/GID `666` and `666`.

You also need to create a consul config directory `/etc/consul` and a data directory `/var/lib/consul` which belong to this user and groups. If you go with installing consul in a container, then you will need to mount these directories inside the image.

Then you must create a configuration file for consul, which should be built off the [docs](https://www.consul.io/docs/agent/options) in a json document. Something like
```json
{
  "datacenter": "polytech",
  "data_dir": "/var/lib/consul",
  "log_level": "INFO",
  "node_name": "<your node name>",
  "server": true
}
```

You will need to add how many bootstrap servers are needed, which IPs to join and so on and so forth to make it work, I would also recommend you use *templates* to get the IP you advertise on your node. This way the same playbook could be run on all the nodes and still make everyhting work. You can also alternatively set all the IPs of all the bootstrap servers in an array variable in your `host_vars`.

Once this is done, pick from one of the following options:

#### Not as a docker container
You will have to download the Consul binary, put it somewhere it can be executed, craft a systemd unit file to start it and so on. It is more interesting but also a bit more work.

:information_source: You will get bonus points for doing it because Xavier hates `systemd`.

#### As a docker container
Follow the documentation to create a docker container using ansible available [here](https://docs.ansible.com/ansible/latest/collections/community/docker/docsite/scenario_guide.html). The image you want to use for this part is the official one available [here](https://hub.docker.com/_/consul).

:warning: to save time you can run docker in the `host` networking mode to avoid having to remap all the ports manually.

### Help your colleagues that are not at this point yet
Just do, it'll be better for everyone else

Once this is done, you should be able to connect to the consule UI and see all the nodes showing up.

### Advertise a service
Have a look at the [consul documentation](https://learn.hashicorp.com/tutorials/consul/get-started-service-discovery) to learn how to write a service, it essentially boils down to creating a json file in the config directory (you might need to pass the `-config-dir` flag to consul, you can have a bunch of json service definitons in the same folder).

Make sure the service has a unique name amongst the groups.

You can start any HTTP service you want, whether it is an nginx or something a bit more complicated, just expose it as a consul service on the correct port

Call someone to check it works, and you're done !
