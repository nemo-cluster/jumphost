This playbook sets up our login policy.

usage:

```shell
make

# or
make users
```

Directory structure for the user's keys:

```
users
├── $username
│   ...
│   └── $keyXX.pub
...
└── user_defs.yml
```

Format of `user_defs.yml`:

```
group_core: wheel       # sudo group

users:
- name: Admin           # do not change admin user (only name, if necessary)
  username: admin       # login name, can be changed
  shell: /bin/bash      # defaults to /sbin/nologin, only admin should be able to login
  group: wheel          # sudo group (should only be used for admin user)
  state: present        # present creates, absent deletes user

- name: User One
  username: user1
  state: present

- name: User Two
  username: user2
  state: present

- name: User Three
  username: user3
  state: absent         # delete user

- name: User Four
  username: user4
  state: present
````