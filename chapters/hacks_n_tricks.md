# Hacks 'n' Tricks

Here I show some hacks or tricks about ansible, that may become handy in certain situations.
As always with hacks, think twice before adopt these. Be sure, you really need it and know, what you are doing.

## The `entrypoint_dir` fact

The `playbook_dir` fact is relative to the playbook, that includes the role/task, that uses this fact.
So, in a case, where you have a subdirectory with playbooks to include, it may happen, that this fact does not contain what you want.

In a case like this, I set a custom fact `entrypoint_dir` in my main playbook (the one, that include the other ones in the subdirectory).
This fact will always contain the path to the main playbook.

```
  site.yml
  hosts: all
  pre_tasks:
    - set_fact:
        entrypoint_dir: "{{ playbook_dir }}"
```

This might be handy, when you have host/group specific files placed outside of your roles but want to address them during your plays.



## Control package state with a variable

You might be uncertain, which package state you should select for your packages: `present` or `latest`.
And this might change during the lifetime of a setup. Think about a play, that should only fix some things, but not update packages. Or you know, that currently the newest version of a package is broken, or, or, or.

To be prepared, You can add a variable, to control the package state:

```
  site.yml
  hosts: all
  pre_tasks:
    - set_fact:
        pgk_state: latest
  tasks:
    - name: ensure packages
      package:
        item: ntp
        state: "{{ pgk_state }}"
    - name: ensure packages (with backup, if the variable does not exist, for whatever reason)
      package:
        item: ntp
        state: "{{ pgk_state|default('present') }}"
```

Now you can control the package state at a global level.
I choosed `present` as the `default`, because it is more defensive and will not do changes, if I forgot the variable.
