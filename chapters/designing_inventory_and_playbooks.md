# Guide for inventory and playbooks

## How to group

Understanding grouping hosts is one of the core concepts for creating scalable inventories and playbooks.

In general, you'll have two types of "configuration layers": A app based, and an environment based.

### app based configuration

App based configuration is everything related to the app. Like "This will need php, nginx and a mysql" to run.

### Environment based configuration

Configuration might change on different environments for a app. Like "for developing, we need xdebug, on production we need an elk-stack for logging"

While app configs may be different for each app, environment configuration might be the same for each app (in general, with exceptions).


## Inventory

The inventory containts two parts:

1. The `hosts`-file: A list of all hosts known by ansible
2. The `group_vars` and (if needed) `host_vars`: Configuration values for each group (and, if needed, for each individual host)

For this setup, we will have multiple inventories, grouped by environment (like `test`, `stage`, `prod`).
So, each inventory contains all hosts for one environment.

Inside the `hosts` file you define the hosts for environment. This could be a static or dynamic list.



### Avoid `host_vars` until you really need it.

`host_vars` let you add configuration for a single host. In our case that is very rare. Instead, we want to configure a group of hosts.
Otherwise it will be very hard to add a new instance for a service. You would have to doublicate the entiry configuration for the host.
Define your projects/services as groups, use `group_vars` to configure them and you can add new instances by simply adding it to the group.

In general, there should be no reason to add host-specific configuration. But if you encounter such a situation, you are still able to add it.
But, if you need to add the same configuration for more than one host, you should consider moving them into a group.


## Playbooks

While the environment is Environment-base on the first level and app based on the second level, playbooks are inversed. You create a playbook for each app and inside the playbook you split it into the different environments: You start with a section, that applies averything, that should be on each environment. Specifiy the app-group name for the `hosts` key. Then you define the environment specific plays by adding the environment-group to the app-group with an ampersand `&`.


For a simple PHP App the reslt playbook can look like this:

```yaml
# playbooks/my-app.yml
---
- hosts: php-app
  roles:
    - role: packages/php
      become: true
    - role: packages/nginx
      become: true
    - role: packages/phpfpm
      become: true
```

One, you have this, you can add the environment related configurations in the same file:

```yaml
# playbooks/my-app.yml
---
- hosts: php-app
  roles:
    - role: packages/php
      become: true
    - role: packages/nginx
      become: true
    - role: packages/phpfpm
      become: true

- hosts: php-app:&develop
  roles:
    - role: packages/phpdev

- hosts: php-app:&live
  roles:
    - role: packages/elk-stack
```

Or, if you know that EACH(!) machine an environment will use the same role, you can create a playbook for this environment.

```yaml
# playbooks/live.yml
---
- hosts: live
  roles:
    - role: packages/ssh
      become: true
    - role: packages/locale
      become: true
```

