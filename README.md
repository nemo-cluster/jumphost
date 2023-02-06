# Installing a Jumphost

This guide is for Alma- and RockyLinux 8 (or CentOS Stream 8).

## RockLinux 8 Installation on the bwCloud

In the bwCloud you can create an instance based on RockyLinux 8. Choose a small flavor, because this machine does not need much RAM or hard disk, e.g. flavor "tiny" with 1GB RAM. You will need an IP that can be reached worldwide, e.g. `132.230.x.y`. You also need an easy-to-remembe host name for the machine.

Login to machine:
```bash
ssh 132.230.x.y -i ~/.ssh/id_rsa-bwcloud -l rocky
```

If you have installed the operating system itself in another location, you must use the user you created.

As a first step, you should update all packages and reboot:
```bash
sudo yum -y update
sudo reboot
```

### Install Ansible Packages

After reboot, install package dependencies.

Login to machine:
```bash
ssh 132.230.x.y -i ~/.ssh/id_rsa-bwcloud -l rocky
```

Install `epel-release` for extra packages:
```bash
sudo yum -y install epel-release
```

Install some dependencies:
```bash
sudo yum -y install ansible git
```

### Creating a User Management Template

If you want to use Github for user and key management, use this template.
https://github.com/nemo-cluster/jumphost

Log in to your account and click the "Use this template" button. On Github, you can create a private repository. If you need access to your repository from your jumphosts, simply add your public keys from the machines to the repository's deploy keys. Go to your repository and select "Settings -> Deploy keys -> Add deploy key". Do **NOT** check the "Allow write accesss" option for your jumphosts!

Open a new terminal on your local desktop and clone the newly created repository:
```bash
git clone https://github.com/<user>/<myrepo>
```

If you don't want to use Github, you can simply clone the repository locally.

Go to your local repository.
Enter the `usermgmt` directory:

```bash
cd <myrepo>/usermgmt
```

Edit users in `vars/mail.yml`.

First remove all dummy users `user1-4`.

> **WARNING:** DO NOT DELETE THE ADMIN USER!

Since you are the admin, remove all dummy keys from admin user and add your key(s), e.g.:
```yaml
users:
- name: Admin           # do not change admin user (only name and username, if necessary)
  username: admin       # login name, can be changed
  shell: /bin/bash      # defaults to /sbin/nologin, only admin should be able to login
  group: wheel          # sudo group (should only be used for admin user)
  state: present        # present creates, absent deletes user
  # add one or more keys
  key:
    - 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAB... joecool@home'

- name: Joe Cool
  username: joecool
  state: present
  key:
    - 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAB... joecool@home'
```

If you want, you can already add the keys of your colleagues yourself, or they can do it themselves afterwards.

Commit and push your initial changes.

### Configure your Jumphost

On your jumphost become `root`:

```bash
sudo -i
```

Check out your repository on the jumphost:
```bash
git clone https://github.com/<user>/<myrepo>
```

Enter repo:
```bash
cd <myrepo>
```

> **WARNING:** DO NOT LOG OUT AFTER ANSIBLE PLAYBOOK!
> Ansible changes sudo rights and removes standard OS user (rocky, almalinux, centos).
> First, check if everything is OK!

Run full Ansible playbook:
```bash
ansible-playbook jumphost.yml
```

You can also select individual roles or disable them:

```bash
ansible-playbook --tags update,ssh jumphost.yml
ansible-playbook --skip-tags osuser jumphost.yml   # skip deletion of OS user (rocky, almalinux, centos)
```

> **WARNING:** Please use a new terminal for the next steps, DO NOT LOG OUT!

Check if the `admin` login works:

```bash
ssh 132.230.x.y -i ~/.ssh/id_rsa-jumphost -l admin
```

Check if `admin` has sudo rights:
```bash
sudo -i
```

Check if you can use your default user for the ssh jumphost (not admin):
```bash
ssh 132.230.x.y -i ~/.ssh/id_rsa-jumphost -l joecool   # should not work
ssh -J joecool@132.230.x.y <finalhostuser>@<finalhost>  # should work
```

The setup is complete. The next steps should be to set up a second jumphost and add your jumphosts as a single SSH entry to your firewalls.

### Configure your local SSH Client

It is best to use multiple SSH keys and configure your SSH client for each host.

Example:

```ssh-config
Host *
    ServerAliveInterval 60
Host jumphost*.subdom1.uni-freiburg.de
    User joecool
    IdentityFile ~/.ssh/id_rsa-jumphost
Host login*.subdom1.uni-freiburg.de
    User loginuser
    ProxyJump jumphost1.subdom1.uni-freiburg.de
    IdentityFile ~/.ssh/id_rsa-login
Host server*.subdom1.uni-freiburg.de
    User root
    ProxyJump jumphost1.subdom1.uni-freiburg.de
    IdentityFile ~/.ssh/id_rsa-server
Host *
    User defaultuser
    IdentitiesOnly yes
    IdentityFile ~/.ssh/id_rsa-default
```

## Ansible Roles

The playbook `jumphost.yml` has the following roles:

```yaml
    - { role: epel-repo,     tags: core     }
    - { role: usermgmt,      tags: core     }
    - { role: sudo,          tags: core     }
    - { role: autoupdate,    tags: update   }
    - { role: ssh,           tags: ssh      }
    - { role: fail2ban,      tags: fail2ban }
    - { role: extra-tools,   tags: extra    }
    - { role: delosuser,     tags: osuser   }
```

For user management to work you must at least run the roles tagged with `core`.

The playbook `jumphost.yml` has an update function that you can omit if you want. But usually it is not a bad idea to update the packages frequently (see role [`autoupdate`](#autoupdate)).

### epel-repo

The role installs the [Extra Packages for Enterprise Linux (EPEL)](https://docs.fedoraproject.org/en-US/epel/) repository. It is required for Ansible and should have been installed in an earlier step.

### usermgmt

Creates users on the host and adds SSH keys. It is mandatory to modify the file `usermgmt/vars/main.yml`.

Example for `usermgmt/vars/main.yml`:

```yaml
---
users:
- name: Admin           # do not change admin user (only name, if necessary)
  username: admin       # login name, can be changed
  shell: /bin/bash      # defaults to /sbin/nologin, only admin should be able to login
  group: wheel          # sudo group (should only be used for admin user)
  state: present        # present creates, absent deletes user
  # add one or more keys
  key:
    - 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAB... user1-home'
    - 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAB... user1-work'
    - 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAB... user2-laptop'

- name: User One
  username: user1
  state: present
  key:
    - 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAB... user1-home'
    - 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAB... user1-work'

- name: User Two
  username: user2
  state: present
  key:
    - 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAB... user2-laptop'

- name: User Three
  username: user3
  state: absent         # delete user
  key:

- name: User Four
  username: user4
  state: present
  key:
    - 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAB... user3-id_rsa'
```

### sudo

Adds the group defined in `sudo/vars/main.yml` to the sudo file. Users in this group should be able to use `sudo`.

> ***WARNING:*** ONLY ADD ADMIN USER TO THIS GROUP!

### autoupdate

Enables the automatic update of the system. Since the host has minimal installation, you can upgrade all packages without any problems. If you want to ensure that Ansible is not upgraded and only security updates should be installed automatically, change `upgrade_type` to `security`.

### ssh

This role does three things:

* Enables `ClientAliveInterval` so that the connection is not lost when used with `sshuttle` for example.
* Disables `root` login: `PermitRootLogin no`
* Disables SFTP connections: `Subsystem sftp`

### fail2ban

Currently, the `fail2ban` role only installs and starts `fail2ban`.

### extra-tools

This role installs some additional tools. You can add your own tools here. However, you should not install too many tools as you should only use this host as a jump host. At the time of writing, the following tools are installed:

* bash-completion
* fish
* git
* htop
* vim

### delosuser
This role removes the default user that comes with many cloud images such as almalinux, rocky, or centos. Since you want to have only one user with `sudo` privileges, remove this user. If the user is not found, nothing happens.
