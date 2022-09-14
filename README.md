# Installing a Jumphost

This guide is for Alma- and RockyLinux 8 (or CentOS Stream 8).

## RockLinux 8 Installation on the bwCloud

On the bwCloud you can create an instance based on RockyLinux 8. Select a small flavor, since this machine does not need mur RAM or disk, e.g. tiny with 1GB of RAM. You will need a IP which is accessible world-wide, e.g. `132.230.x.y`. You will probably need a good host name as well.

Login to machine:
```bash
ssh 132.230.x.y -i ~/.ssh/id_rsa-bwcloud -l rocky
```

If you have installed the OS somewhere else, you can use this user for login.

You can already update all packages and reboot:
```bash
sudo yum -y update
sudo reboot
```

### Install Ansible Packages

After reboot install dependent packages.

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

If you want to use Github, use this template.
https://github.com/nemo-cluster/jumphost

Just click on "Use this template".

Open a new terminal and clone your new repo to your local desktop:
```bash
git clone https://github.com/<user>/<myrepo>
```

If you do not want to use Github you, you can just clone the repo locally.

Go to your local repository.
Enter the `usermgmt` directory:

```bash
cd <myrepo>/usermgmt
```

Edit users `vars/mail.yml`.

First remove all dummy users `user1-4`.

> **WARNING:** DO NOT DELETE THE ADMIN USER!

Since you are the admin, remove all dummy keys from admin user and add your key(s), e.g.:
```yaml
users:
- name: Admin           # do not change admin user (only name, if necessary)
  username: admin       # login name, can be changed
  shell: /bin/bash      # defaults to /sbin/nologin, only admin should be able to login
  group: wheel          # sudo group (should only be used for admin user)
  state: present        # present creates, absent deletes user
  # add one or more keys seperated with newline
  key:
    - 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAB... joecool@home'

- name: Joe Cool
  username: joecool
  state: present
  key:
    - 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAB... joecool@home'
```

If you want you can already add the keys of your co-workers yourself, or they can do it themselves afterwards.

Commit and push your initial changes.

### Configure your Jumphost

On your jumphost become `root`:

```bash
sudo -i
```

Checkout your repository:
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

Run Ansible:
```bash
ansible-playbook jumphost.yml
```

You can select roles or disable them:

```bash
ansible-playbook --tags update,ssh jumphost.yml
ansible-playbook --skip-tags osuser jumphost.yml   # skip deletion of OS user (rocky, almalinux, centos)
```

> **WARNING:** Please use a new terminal for the next steps, DO NOT LOG OUT!

Check if the admin login works:

```bash
ssh 132.230.x.y -i ~/.ssh/id_rsa-jumphost -l admin
```

Check if admin has sudo rights:
```bash
sudo -i
```

Check if you can use your standard user for the ssh jumphost:
```bash
ssh 132.230.x.y -i ~/.ssh/id_rsa-jumphost -l <user>   # should not work
ssh -J <user>@<jumphost> <finalhostuser>@<finalhost>  # should work
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

For the user management to work you need to run at least the roles tagged with `core`.

### epel-repo

Role installs the [Extra Packages for Enterprise Linux (EPEL)](https://docs.fedoraproject.org/en-US/epel/) repository. It is needed for Ansible. Should already be installed in an earlier step.

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
  # add one or more keys seperated with newline
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

Adds group in `sudo/vars/main.yml` to sudo file. Users in this group should be able to use `sudo`.

> ***WARNING:*** ONLY ADD ADMIN USER TO THIS GROUP!

### autoupdate

Enables auto update of the system. Since the host has only a minimal installation you can update all packages without any problems. If you want to make sure, that Ansible does not get updated and want only security updates to be installed automatically change `upgrade_type` to `security`.

### ssh

This role does three things:

* Enables `ClientAliveInterval`, so that your connection does not get lost if used with `sshuttle`.
* Disables `root` login: `PermitRootLogin no`
* Disables SFTP connections: `Subsystem sftp`

### fail2ban

Currently the role `fail2ban` only installs and starts `fail2ban`.

### extra-tools

This role installs some extra tools. You can add your own tools here. But you shouldn't install to many tools, since you should use this host only as a jump host. In the time of writing the following tools are installed:

* bash-completion
* fish
* git
* htop
* vim

### delosuser

This role removes the standard user, which comes with many cloud images like almalinux, rocky or centos. Since you only want one user with `sudo` rights, remove this user. If the user is not found, nothing happens.

