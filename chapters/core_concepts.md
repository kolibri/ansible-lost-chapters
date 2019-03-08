## What is Ansible, and what is it for

- A provisioning tool
- Written in Python, configured/used with YAML files
- Used for automatic setup of any kind of system (as long it's Linux, MacOS and Windows are supported, but limited by the nature of the OS #osBashing)


## What is the benefit of a scripted infrastructure

- It is documentated
- It becomes discussable
- It becomes repeatable
- It becomes testable
- It helps a lot with scaling

## Things you should know before starting

### Usual idea of usage

You got one "control machine", that has ansible installed, and several other machines, that should be managed by ansible.
the managed machines do not need ansible installed, the only requirement for them is python (>=2.6 or >=3.5), which is often already present (and the possibility to connect to this machine)

```
               ┌----┐
               | CM |
               └----┘
                ||||
    ┌-----------┘||└-----------┐
    |        ┌---┘└---┐        |
    |        |        |        |
    v        v        v        v
  ┌----┐   ┌----┐   ┌----┐   ┌----┐
  | M0 |   | M1 |   | M2 |   | M3 |
  └----┘   └----┘   └----┘   └----┘

CM: Control machine
M0-3: Managed machines
```

### Every ansible YAML file is a Jinja2-Template

(You already know this template engine. It the mother of Twig.)


## Core concepts of Ansible

### Inventory

The inventory is a file (or a collection of files), where we define our target machines.
This could be VMs, desktop-pc's, EC2 instances, Docker containers and so on, as long it can be accessed by ansible (usually via SSH, but there are more options).
we can (and should) group our hosts (like "all database server", "all webservers", etc.)

Also part of the inventory (but NOT the inventory file) is the special configuration of each host or group. This is placed inside YAML filed inside the `host_vars`/`group_vars` directory.


### Task

A task is a single step during the provisioning. It can be theoretically anything, as long a module exists for the task.

Some examples (with the belonging module-name)
- Ensure that file XXY exists with this content (`template`, `copy`)
- Ensure service ABC is running (`service`)
- Ensure user FOOBAR exists and has no sudo privilege (`user`)
- Deploy APP to PROD (`deploy_helper`)
- Run script TAKE_WORLD_LEADERSHIP (`shell`, `command`)
- Ensure this git repo is checked out and at the current commit of that branch (`git`)

There are a lot built in modules to explore, that should satisfy 99% of the usual needs.

Tasks are written in "Playbooks" or "Roles". You can also execute tasks directly with the `ansible` command line tool (but this is rarely used and not recommended)


### Handler

A task, that is executed at the end of a run, if another tasks notifies the handler to run.
Mostly used for restarting services after their configuration has changed.


### Playbook

A playbook is that part, that is executed by the `ansible-playbook` command.

It defines, on which host which tasks/roles should be executed.


### Role

The concept of resuability and structuring in ansible.

A role is a collection of tasks and the belonging handlers, templates, files and so on.
A good role is like a good class in OOP: It solves one problem, has everything it needs, nothing else.

For example, you can define a `php` role, that ensures that php is installed on your system.
It ensures all the related packages, the special configuration for your needs, it could ensure the php-fpm-service is running, etc.

Roles can contain configuration. They should bring their own default configuration, that can be overwritten inside the inventory configuration.

Roles can depend on each other (e.g. the `php-fpm` role can depend on the `php` role)

You can write roles, that support multiple OS.

It is best practice to define all your infrastructure inside of roles.

There is a huge collection of roles on (https://galaxy.ansible.com/) you can already use, but I do not recommend to do it.
I guess, you want to define your infrastructure by your own and do not review every change that happens in third party roles 
depending on a third party role without reviewing it could lead to security issues)

Directory structure of a role:

```
my_awesome_role
├── README.md
├── defaults
│   └── main.yml  # contains all default configuration
├── files         # contains all files, that can be uses (e.g. by the `copy` module)
├── handlers     
│   └── main.yml  # contains all handlers for the role
├── meta
│   └── main.yml  # contains meta information (like the autor) and also the dependencies to other roles
├── tasks
│   └── main.yml  # the main file of the roles. Place your tasks here
├── templates     # contains the template files (all templates are Jinja2 templates and should hav ethe file ending `*.j2`)
├── tests         # you can add tests for roles, but this is rarely used
│   ├── inventory
│   └── test.yml
└── vars          # place vars for the role here
    └── main.yml
```

### Facts

Facts are informations about the managed nodes provided by ansible.
I reccomend to start every new playbook with the task `- debug:  var=hostvars`, which shows all known facts about the current system.
Take some time and get familiar with this list.
You should not know every information it provides, but have a guess, if you should find a certain information inside facts or not.
You should know, that you can use all these variables inside your playbooks/roles/tasks.

