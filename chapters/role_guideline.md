# Guidelines for roles

## THERE IS A MODULE FOR THAT!

https://docs.ansible.com/ansible/latest/modules/list_of_all_modules.html

If module exist, use it! Or have a good excuse for not using it.

## THERE IS A FACT FOR THAT

```yaml
# playbook.yml

- hosts: all
  tasks:
    - debug: var=hostvars
```

Run this playbook and see the huge amount of facts ansible provides. If your information is inside, use it.
No need to call e.g. `uname` via shell module, register the variable and so on.

This also means, that we should not hardcode os specific variables (like the release name from ubuntu inside apt repositories)

## Role variables

Prefix role variables with the name of the role to avoid naming conflicts.


## The different types of roles

You can specify different types of roles by grouping them into subdirectories inside `roles`.
For example there are package related roles, that handle single "packages" like nginx, php and so on.
Then there are project related roles, that may consume the package related roles.
(e.g. the role for a symfony application could depend on the php, nginx and mysql role.)

Another type of roles is described in the chapter [Module Roles](./module_roles.md)

Example directory structure

```
roles
├── modules
│   └── aur
├── project
│   └── my_application
└── packages
    ├── ansible
    ├── mysql
    └── php
```

## You can use a role more than one in a play

Roles are not limited to one usage. You can add them several times to your plays and they will be executed each time.
This might be useless, but think about a role, that setups one user account on your machine.
You can use this role multiple times to add multiple accounts by calling the role with different values each time.

This is simmilar to [Module Roles](./module_roles.md) (but will differ in the way you call the roles)

Example for a playbook.

```
# site.yml
- hosts: all
  roles:
    - role: user_account
      vars:
         username: Torben Tester
         password: not_so_safe
    - role: user_account
      vars:
         username: Tamara Tester
         password: better_but_still_unsecure
```

## Password

Passwords do not belong into roles. Define the variable as a default, but define it with null (`~`).
To make sure we use a password, add this little check to your role task:

```yaml
- assert:
    that:
      - my_role_password is not none
```

## Templates have the file extension `*.target_format.j2`

Templates should have two file extension: `my_template.target_format.j2`.
`target_format` should be replaced with the target file extension (e.g. `.conf` for config files).
The `.j2` is mandatory to indicate, that this is a template.
So, for a PHP file a valid template filename is: `my_php_file.php.j2`.

## Do not use inline style

Inline style is hard to read. So, avoid/ban it. The only exception from this is, when you only got one argument for the task:

```yaml
# this is good
- name: ensure foobar
  package:
    name: foobar
    state: present

# this is valid
- name: ensure foobar 
  package: name=foobar

# this not. Don't do this
- name: ensure foobar 
  package: name=foobar state=present
```

But keep in mind, that you have to sorround jinja variables with quotation marks:

```yaml
# does the job
- name: ensure foobar
  package:
    name: "{{ package_list }}"

# throws an error
- name: ensure foobar
  package:
    name: {{ package_list }}
```

## Handlers and enabling services

The idea of having handlers is, that a service should only restarted, when it is needed and only once a playbook run.
To archive this, place the `notify` directives only at tasks, that leads to a service restart when the task has changed.
**This includes the installation/update of the package!**

Also the handler itself should do nothing than restarting the service. If you use the `service` module, do not add handling for enabling the service.
Enabling a service (saying, that it should start on booting) is a task.

```yaml
# my_role/tasks/main.yml

- name: ensure foobar is enabled
  service: 
    name: foobar
    enabled: yes
```

The handler just handles the restart:

```yaml
# my_role/tasks/main.yml

- name: restart foobar
  service: 
    name: foobar
    state: restarted
```

## Notifying handlers

The notify takes a list of handlers. So a list should be passed instead of a string:

```yaml
# foobar/tasks/main.yml

- name: ensure foobar is enabled
  package: 
    name: foobar
    state: present
  notify: [restart foobar]
```

## Avoid blocks

Blocks behave strange and unexpected and add some weird indenting behavior.
Try to avoid them. Best solution in most cases is to extract the block-content into a new file an include it.

## variables in `defaults/` vs variables in `vars/`

Thread "vars" like constants. The use case for them is to load environment specific sets of configuration
(think about the package name for apache webserver. In some distros it is called `httpd`, in another `apache` or `apache2`).
Technically, they can be overwritten by configuration, but you should not do this (remember OOP in PHP4!)

"defaults" are the place for role configuration that can be changed by the user of the role.

*If uncertain, place the variable as default!*

A role should only use variables, that are defined inside the role.
There are two exceptions: You may use facts provided from ansible if it makes sense.
For example the `os_family` fact to handle different os-types.
But be careful: For example the direct usage of `ansible_user_id` (which contains the username of the user that ansible uses) would limit the role to this username.
In this case it is better to create a variable for the role and predefine it with the `ansible_user_id`. With this, you can still use another username for this role.

The second exception can be a variable from a role, that is a dependency to the current role.
Be sure, you added the dependency inside `meta/main.yml`.

All variables used in a role should be defined inside `defaults/main.yml` with a propper default value.
If this is not possible (like for credentials) you should define the variable with null (`~` in YAML) and add a `assert` tasks to ensure the variable is set and can be used.
Alternativly you can add `when` conditions to the tasks, that need this variable. But make sure the variable is usable inside your tasks.

It is a good practice to add documentation to the roles variables, so the next developer can easily see what type of content it takes.

## Avoid tags

Tags are a construct for controlling which tasks are executed and which not.
They add an (or even more) extra level of complexity, that has to be maintained and tested rigorous.
I saw tags used for contidional execution (run this, when target is an cloud instance).
The right usage would be a `when` condition based on facts.

Avoid tags as hard as you can. There should be no need to have them.

## Roles are project and environment agnostic.

A role does not know, where it is executed. It does not know anything about your project.
Design your roles in a way, that it does not use the environment or project name.

If a role should be able to behave different on different environments, first think about, if it still is ONE role,
that you are designing, or if it is better to create two (ore more) roles.

If you decide, that it is one role, don't react to the environment. Add variables, that are describing and environment agnostic instead.

Examples:

### On develop machines, XXY should be installed, on production it may not be installed. (Or vice versa)

*Don't* add a `when: env == dev` condition to the XXY-related tasks!
Instead, , add a flag `install_XXY` to the role. Don't forget to define it inside the `defaults/main.yml`.

### The XXY-service needs a different vhost-configuration for nginx

*Don't* add a `when: app == XXY-service` or some kind of "smart template finding" based on the `app` variable. (Or how your variable is named)
Instead make the template name configurable and specify the template in the inventory configuration (`host_vars`/`group_vars`).
Don't forget to add a default value inside `defaults/main.yml`

With this, you will get roles, that are easy to understand, as you already see inside at the `defaults` configuration, what it can do.
And without relying onto the environment, you are able to add new environments without touching the roles.

How the role behaves is now configurable for each environment at inventory-level.
(It's the "Inversion of control"-pattern applied to ansible-roles).

