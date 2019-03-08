## Common pitfalls

## Idempotency

Ansible playbooks have to be [idempotent](https://en.wikipedia.org/wiki/Idempotence).
That means, even if you run the playbook a lot of times on the same node, the result will always be the same, as if you run the playbook only once.
This is a core dependency and ensures reliability of the plays.

Ansible (and any other provisioning tool) does not install software, add configurations, etc, it *ensures that the node has the described state*.
This may look just as a different wording, but it makes a difference on how you see provisioning.

**You describe the state, that the node should have after the playbook run.**

It is a good practice to take over this behaviour while naming tasks: Don't name them "install nginx". Name them "ensure nginx is installed" or "nginx is installed".

You have to take care, that all roles are idempotent. In most cases, the modules are already designed in that way.
But there are some exceptions. For example, when you use the `shell` module to create a file, you have to add a `creates` option to let ansible know when the command should be executed.
Another directive is the `changed_when` option.

A simple test for idempotency: Execute the playbook twice. On the second run inside the summary of the run should note, that nothing changed:

```
PLAY RECAP ***************************************************************************
node_name                           : ok=221    changed=0    unreachable=0    failed=0
```


### Nested role configuration is not good.

When writing default configuration for your roles, you might come to the idea of writing it in this way:

```yaml
user:
  name: my_user
  home: /home/my_user
```

It may look nice, but you will get issues when overwriting certain configurations.
Also you cannot reuse variables inside the same file.
Instead, write configuration like this:

```yaml
user_name: my_user
user_home: "/home/{{ user_name }}"
```

Now you can easily overwrite the `user_name` variable in your inventory configuration.
Also you already see, that you can reuse variables. With setting the `user_name` you now also specify the `user_home`.

## Variables are global

All variables inside a playbook run are global. You can access variables from other roles (as long they are part of the playbook run) without restrictions.
This means, you also can overwrite them at any place and add some debugging headache to the next developer (wich is always your future-self).
Saying this, I recommend to prefix all variables with the name of the role they live in.
Variables, that live outside of roles should either be so self speaking or prefixed with something else (the vendor name maybe. I have no recommenation except to avoid variables outside of roles)

## Inline style of tasks

If you look for examples in the modules documentation, you will in most cases see the inline style of tasks:

```yaml
- name: ensure root privileges for user
  lineinfile: dest=/etc/php/7.2/php.ini state=present regexp="^#?\w*date.timezone.*" line="date.timezone=Europe/Berlin"
```

As you see, this is not that readable.
Instead consequently use the "real YAML" format. (You may make exceptions if the tasks contains only one short argument.)


```yaml
- name: ensure root privileges for user
  lineinfile: 
    dest: /etc/php/7.2/php.ini 
    state: present 
    regexp: "^#?\w*date.timezone.*" 
    line: "date.timezone=Europe/Berlin"
```

You see, much more readable.

### Difference between `defaults` and `vars`

There are two places, where to place configuration variables inside a role: `defaults` and `vars`.
Technicaly, there is no difference between them, once you got a variable you cannot say if it was a "default" or a "var".
But they have different use cases.

#### defaults

"Defaults" are the "public" configuration of your role. Every user of the role should be allowed to overwrite these configs.

Lets say, we have a role, that configures SSH. And you want the user of your role to allow to configure if root users can login with password, which defaults to "no!".
Then you can add a `ssh_allow_root_login_with_pass: false` variable to your configs, and use this variable in your tasks/templates.

#### vars

"Vars" are configuration, that is private. The user technically CAN but SHOULD NOT overwrite these.
They are often used to get configuration based on other facts of your system.
Lets stick with the SSH role: On debian based systems the name of the ssh-service is usually `ssh`. While on archlinux the service-name is `sshd`.
To create a role, that supports both distributions, you can (and should) use the `include_vars` directive.
Place a file for each target system in `vars`, name it like the fact for the distibution name (I like to use `ansible_os_family`).

An example:

```yaml
# roles/ssh/vars/Debian.yml
ssh_service_name: ssh
```
```yaml
# roles/ssh/vars/Archlinux.yml
ssh_service_name: sshd
```
```yaml
# roles/ssh/tasks/main.yml
- name: Include OS-Specific variables
  include_vars: "{{ ansible_os_family }}.yml"

- name: enable ssh service
  service:
    name: "{{ ssh_service_name }}"
    enabled: yes
```

Depending on the `ansible_os_family` fact the correct vars-file is loaded and the provided `ssh_service_name` variable contains the corrent value to enable the service.

**If you are uncertain, where to put a variable, put into `defaults`**

### Understanding `become`/privilege escalation

The concept of `become` might be a bit confusing at the first time. But it is simple, trust me.
First thing you have to know: Every tasks is executed by the user, that is used to connect to the managed machine.
So, if you connect via ssh with the user `vagrant`, every tasks will be executed by the `vagrant` user.
But, what if you need root privileges, like for installing packages, creating users and so on?

This is the place, where `become` comes in places.
It is a mechanism to switch the current user and "become" another one, that (hopefully) has the priviliges to do the task.

Second thing to know is, that the become mechanism is built with four directives you can place at tasks, roles, includes, etc.
The four are:

- `become`: toggles privileges escalation. **defaults to `false`**
- `become_user`: defines the user to become. **defaults to `root`**
- `become_method`: defines the method to become the user. **defaults to `sudo`**
- `become_flags`: flags for becoming another user


The strong part about `become` are the default values. In most cases you just want to gain root privileges. So it is enough to add `become: true` to you task/include/role/etc.
Because of the default values, the worker becomes  the `root` via `sudo` user and can perform the task.

Example:

```yaml
- name: Install some packages
  become: true
  package:
    name: php
```

This implies the following:

```yaml
- name: Install some packages
  become: true
  become_user: root
  become_method: sudo
  become_flags: ~   # Is not needed here
  package:
    name: php
```

This hits about the following: EVERY tasks is executed with `become`, `become_user`, configured etc.
But as `become` is `false` per default, the privilege escalation does not apply to tasks, where no `become` is configured.
This is also the reason, why you have to configure `become: true`, even if you defined the `become_user`.
*Setting the user, method or flags does NOT imply `become: true`.*



### Restarting services at task level is not what you want

When you change configuration for a service (like nginx, php-fpm, systemd, etc) the service often needs to restart to recognize these changes.
The first approach would be, to use the `service` module inside your tasks and just restart it.
But this has some issues: At first, you cannot predict, if later during the run on another change for the service is made.
You could place another "restart"-Task right after every change, that needs a restart, but that would lead to doublicated code and unnesessary restarts of the service.
*The goal is, to restart every service only, when it is needed and only once per run*
You know, restarting nginx/fpm can be dangerous as it might cancel all current user sessions.

To get rid of this, you *do not restart* services at task level. The only purpose of the `service` module at task level is to *enable* a service.
You need to place a handler
