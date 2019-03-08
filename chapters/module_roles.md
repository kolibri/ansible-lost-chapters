# Module roles (aka: including roles)

Ansible has a lot of builtin modules, and they should satisfy 99% of all our needs.

## The `aur` example

But, there a some exceptions. For example there is no role to install aur packages for archlinux.

Defining the steps to automate this and loop over these steps with a list of package names might work,
but it's not the ideal case, because it forces me to collect all aur packages to install at one variable.

I want something, that feels mor like the `package` module.

This is, where the (I call them) "module roles" come into place.

What makes them different to other roles:

- They cannot be run without arguments (I need to know, wich package I want to install)
- They are designed to be included multiple times
- They (usualy) not included inside the `roles` level in playbooks, but in the `tasks` of other roles.

Lets have some code:

```
# roles/module/aur/tasks/main.yml
---
- assert:
    that:
      - pkg_name is defined
      - makepkg_nonroot_user is defined


- name: AUR | get metadata from AurJson api
  uri:
    url: "https://aur.archlinux.org/rpc.php?type=info&arg={{ pkg_name | mandatory }}"
    return_content: yes
    timeout: 6
  register: api_info

- assert:
    that:
      - api_info.status == 200
      - api_info.json is defined
      - api_info.json.type == 'info'
      - api_info.json.resultcount == 1
      - api_info.json.results is defined

- name: AUR | get installed package version
  shell: "pacman -Q | grep {{ pkg_name }} | cut -d' ' -f2"
  register: pacman_query_result

- name: AUR | {{ pkg_name }} | get package git repo
  become: true
  become_user: "{{ makepkg_nonroot_user }}"
  git:
    repo: "https://aur.archlinux.org/{{ pkg_name }}.git"
    dest: /tmp/{{ pkg_name }}
  when: api_info.json.results.Version != pacman_query_result.stdout
  register: extracted_pkg

# This will break if run as root. Set user to use with makepkg with 'makepkg_user' var
- name: AUR | {{ pkg_name }} | build package, including missing dependencies
  become: true
  become_user: "{{ makepkg_nonroot_user }}"
  command: makepkg --noconfirm --noprogressbar -mfs
  args:
    chdir: /tmp/{{ pkg_name }}
  when: extracted_pkg | changed
  register: aur_makepkg_result

- name: AUR | {{ pkg_name }} | install newly-built aur package with pacman
  become: true
  shell: pacman --noconfirm --noprogressbar --needed -U *.pkg.tar.xz
  args:
    chdir: "/tmp/{{ pkg_name }}"
  register: pacman_install_result
  when: aur_makepkg_result | changed
  changed_when: pacman_install_result.stdout is defined and pacman_install_result.stdout.find('there is nothing to do') == -1

```

It is not important, to understand everything, that is going on here.

Note the first `assert` block. It ensures, that `pkg_name` and `makepkg_nonroot_user` are defined! Otherwise ansible will throw an error.

Lets have a look, how we can call this role to install mondogb.

```
# roles/packages/mongodb/main.yml

- hosts: all
  tasks:
    - name: install aur packages
      include_role:
        name: aur
      vars:
        pkg_name: mongodb
        makepkg_nonroot_user: "{{ ansible_user_id }}"
      when: ansible_distribution == "Archlinux"

```

Not the same as a built in module, but not that far away, I guess.

This become handy, when you design roles, that should work on archlinux and another distribution, that maybe has mongodb already in its repositories.


## The `php.ini` example.

There is another scenario, where this pattern (in a variation) may become handy: When you got a package, that has configuration, but this configuration depends on the project.
Or other packages can add configuration/register to a service, etc.

Lets take the `php.ini` file. The role for ensuring php is installed may provide some defaults,
but cannot know, what specific settings the application needs.

So, lets create a tasks file, that handles changes in the `php.ini` file.

```
# roles/packages/php/tasks/php_ini.yml
- assert:
    that:
      - env is defined
      - settings is defined

- name: ensure php.ini settings for {{ env }}
  lineinfile:
    dest: "/etc/php/{{ php_version }}/{{ env }}/php.ini"
    regexp: "^;?\\s*{{ ini.key }} ="
    line: "{{ ini.key }} = {{ ini.value }}"
  loop: "{{ settings|dict2items }}"
  loop_control:
    loop_var: ini
```

Again, note the `assert`, that ensures the needed variables exist.
Also note, that this is not the `tasks/main.yml`.

Lets use this file:

```
# roles/packages/php/tasks/main.yml
- name: ensure php ini settings for cli
  include_role:
    name: packages/php
    tasks_from: php_ini
  vars:
    env: cli
    settings: "{{ php_ini_cli_settings }}"
```

This now is the `tasks/main.yml`, that is executed, when you normaly include the role.
The key directive here is `tasks_from` from the `include_role` module.

You can now use this pattern to ensure settings for fpm inside the `php_fpm` role:

```
# roles/packages/php_fpm/tasks/main.yml
- name: ensure php ini settings for fpm
  include_role:
    name: packages/php
    tasks_from: php_ini
  vars:
    env: fpm
    settings: "{{ php_fpm_ini_cli_settings }}"
```

Yes, it's (almost) the same :)


## Other use cases for this pattern

- Nginx/Apache vhosts
- creating databases for applications
- adding users to a system
- ...


