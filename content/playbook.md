---
draft: false
genres:
- ansible
tags:
- playbook
- ansible
menus: playbook
weight: 1
title: Structuring your Ansible Playbook
---


***

## Introduction

***

The automating of tasks with Ansible begins and ends with tasks defined in a playbook. There are various ways to structure your playbook, including:

   - Defining everything in a single playbook 
   - Defining tasks across multiple playbooks
   - Using a single playbook to include roles that contain various organised tasks

For me, I prefer the latter option. I have a single `playbook.yml`, and various roles containing tasks to automate the deployment of my docker services. However, there is no right way of doing things, and it's simply a case of how many tasks you're dealing with, what sort of tasks, and simply how you prefer to do things.

My `playbook.yml` is structured to include the following:
   - Hosts
   - Pre_Tasks
   - Tasks
   - Handlers

And calling the playbook is as simple as:

```cli

ansible-playbook play.yml -i hosts.ini --ask-become-pass --ask-vault-pass --tag arrs

```

The rest of this document will break down each section of my playbook.


***

## Hosts

***

This section defines the hosts you want the playbook to run against:


```yaml

- hosts: skynet
  become: true

```

As I'm working with multiple hosts, I have them in a group named 'skynet' with this group defined in a `hosts.ini` file in the same directory as the `playbook.yml`:

```ini
 [skynet]
 localhost ansible_connection=local ansible_user=redacted
 plex ansible_host=redacted ansible_user=redacted
 saltbox ansible_host=redacted ansible_user=redacted
```

Of course, if you're just a single machine user you can do away with the group:

```yaml

- hosts: localhost
  become: true

```

***

## Pre_Tasks

***

Pre-tasks, as the name suggests, always run first during a play. I reserve these for tasks that I will need for all or the majority of my roles:


```yaml

  pre_tasks:
    - name: Gather packages
      ansible.builtin.package_facts:
        manager: auto
      tags: always

    - name: Gather IP geolocation data
      community.general.ipinfoio_facts:
      tags: always

    - name: Gather public IP data
      community.general.ipify_facts:
        timeout: 20
      register: public_ip
      tags: always

    - name: Set timezone variable
      ansible.builtin.set_fact:
        timezone: '{{ ansible_facts.timezone }}'
      tags: always

    - name: Public IP output
      ansible.builtin.debug:
        msg: '{{ ipify_public_ip }}'
      tags: always

```

In the above, I get package, timezone, and public IP information for each host. These are important when installing packages, setting docker env values, and dealing with DNS tasks.

For example, the below task relies on information gathered by the 'Gather packages' task:

```yaml
- name: Include Docker install tasks
  when: '"docker-ce" not in ansible_facts.packages'
  ansible.builtin.include_tasks: 'sub_tasks/{{ item }}.yml'
  loop:
    - install
    - swarm
```

***

## Tasks

***

The tasks section is the bulk of my playbook, and is simply used to include my various roles using the `ansible.builtin.include_role` module:


```yaml

  tasks:

################################
# UBUNTU
################################

    - name: Ready Ubuntu
      when: inventory_hostname == 'localhost'
      ansible.builtin.include_role:
        name: ubuntu
        apply:
          tags: ubuntu
      tags: ubuntu

    - name: Ready Ubuntu for Plex
      when: inventory_hostname == 'plex'
      ansible.builtin.include_role:
        name: ubuntu2
        apply:
          tags: ubuntu2
      tags: ubuntu2

################################
# DOCKER (PVR MACHINE)
################################

    - name: Conduct docker tasks
      when: inventory_hostname == 'localhost'
      ansible.builtin.include_role:
        name: docker1
        apply:
          tags: docker1
      tags: docker1

################################
# DOCKER (PLEX MACHINE)
################################

    - name: Load plex token variable
      when: inventory_hostname == 'localhost'
      ansible.builtin.set_fact:
        plex_worker_token: '{{ lookup("ini", "plex_token_worker section=" + "docker" + " file=" + "/ansible/vault.ini") }}'
      tags: docker2

    - name: Fetch token variable for plex host
      when: inventory_hostname == 'plex'
      ansible.builtin.set_fact:
        plex_worker_token: '{{ hostvars["localhost"]["plex_worker_token"] }}'
      tags: docker2

    - name: Ready Ubuntu (Plex)
      when: inventory_hostname == 'plex'
      ansible.builtin.include_role:
        name: docker2
        apply:
          tags: docker2
      tags: docker2

    - name: Fetch node name from plex host
      when: inventory_hostname == 'plex'
      ansible.builtin.set_fact:
        plex_worker_node: '{{ ansible_facts["nodename"] }}'
      tags: docker2

    - name: Set worker node label to 'plex'
      when: inventory_hostname == 'localhost'
      docker_node:
        hostname: '{{ hostvars["plex"]["plex_worker_node"] }}'
        labels:
          ansible_host: plex
        labels_state: replace
      tags: docker2

################################
# BLOG
################################

    - name: Deploy blog stack
      when: inventory_hostname == 'localhost'
      ansible.builtin.include_role:
        name: blog
        apply:
          tags: blog
      tags: blog

################################
# GITEA
################################

    - name: Deploy gitea stack
      when: inventory_hostname == 'localhost'
      ansible.builtin.include_role:
        name: gitea
        apply:
          tags: gitea
      tags: gitea

################################
# MARIADB
################################

    - name: Deploy mariadb stack
      when: inventory_hostname == 'localhost'
      ansible.builtin.include_role:
        name: mariadb
        apply:
          tags: mariadb
      tags: mariadb

################################
# POSTGRES
################################

    - name: Deploy postgres stack
      when: inventory_hostname == 'localhost'
      ansible.builtin.include_role:
        name: postgres
        apply:
          tags: postgres
      tags: postgres

################################
# ARRS
################################

  ## Bazarr, Lidarr, 

    - name: Deploy arrs stack
      when: inventory_hostname == 'localhost'
      ansible.builtin.include_role:
        name: arrs
        apply:
          tags: arrs
      tags: arrs

################################
# COMPANIONS
################################

  ## Services: AutoBrr, Doplarr, Jellyseerr, Ombi, Recyclarr, TheLounge, ZNC

    - name: Deploy companions stack
      when: inventory_hostname == 'localhost'
      ansible.builtin.include_role:
        name: companions
        apply:
          tags: companions
      tags: companions

################################
# USENET
################################

  ## Services: Sabnzbd, NZBHydra2

    - name: Deploy usenet stack
      when: inventory_hostname == 'localhost'
      ansible.builtin.include_role:
        name: usenet
        apply:
          tags: usenet
      tags: usenet

################################
# METRICS
################################

    - name: Deploy metrics stack
      when: inventory_hostname == 'localhost'
      ansible.builtin.include_role:
        name: metrics
        apply:
          tags: metrics
      tags: metrics

################################
# DNS
################################

    - name: Deploy dns stack
      when: inventory_hostname == 'localhost'
      ansible.builtin.include_role:
        name: dns
        apply:
          tags: dns
      tags: dns

################################
# TRAEFIK
################################

    - name: Deploy traefik stack
      when: inventory_hostname == 'localhost'
      ansible.builtin.include_role:
        name: traefik
        apply:
          tags: traefik
      tags: traefik

################################
# UNIFI
################################

    - name: Deploy unifi stack
      when: inventory_hostname == 'localhost'
      ansible.builtin.include_role:
        name: unifi
        apply:
          tags: unifi
      tags: unifi

################################
# GLUETUN
################################

    - name: Deploy gluetun stack
      when: inventory_hostname == 'localhost'
      ansible.builtin.include_role:
        name: gluetun
        apply:
          tags: gluetun
      tags: gluetun

################################
# UNIONFS
################################

    - name: Deploy unionfs
      when: inventory_hostname == 'localhost'
      ansible.builtin.include_role:
        name: unionfs
        apply:
          tags: unionfs
      tags: unionfs

################################
# UTILITIES
################################

    - name: Retrieve Plex token
      when: inventory_hostname == 'localhost'
      ansible.builtin.set_fact:
        plex_auth_token: '{{ lookup("ini", "token section=" + plex_name + " file=" + plex_token_location) | regex_replace("\n", "") }}'
      tags: utilities

    - name: Deploy utilities
      when: inventory_hostname == 'localhost'
      ansible.builtin.include_role:
        name: utilities
        apply:
          tags: utilities
      tags: utilities

################################
# PLEX
################################

    - name: Prepare plex token
      when: inventory_hostname == 'localhost'
      ansible.builtin.include_role:
        name: plex
        apply:
          tags: plex
      tags: plex

    - name: Prepare plex stack
      when: inventory_hostname == 'plex'
      ansible.builtin.include_role:
        name: plex2
        apply:
          tags: plex
      tags: plex

    - name: Deploy plex stack
      when: inventory_hostname == 'localhost'
      ansible.builtin.include_role:
        name: plex3
        apply:
          tags: plex
      tags: plex

```

There's nothing really stopping you from having one gigantic playbook, but I prefer to keep related-relevant tasks within roles.


***

## Conditionals

***

Key to the single playbook setup are conditionals, because:

   1) When I run a playbook I typically only want a single role to run (with few exceptions).
   2) The tasks and roles I have are typically written for a single host

Thus, each task within the `playbook.yml` will look like the following:

```yaml

    - name: Ready Ubuntu for Plex
      when: inventory_hostname == 'plex'
      ansible.builtin.include_role:
        name: ubuntu2
        apply:
          tags: ubuntu2
      tags: ubuntu2

```

With the two primary conditionals being the hostname to run-on and tags to include:

***

### Hostnames

***

There use of the `when: inventory_hostname == 'plex'` shown above is simply asking the role to be run on my plex machine. The hostname in this case is what was defined for my plex machine in the `hosts.ini` file:

```ini
 [skynet]
 localhost ansible_connection=local ansible_user=redacted
 plex ansible_host=redacted ansible_user=redacted
 saltbox ansible_host=redacted ansible_user=redacted
```

Defining the hostname here prevents you having to include sensitive information (i.e, tailscale ips and others) in the playbook, and it can be whatever nickname you want it to be.

In a Docker Swarm setup, the use of hosts is important, as you'll often need one host to deploy the services, but you'll need to do tasks on the worker machine to ready it for the service deployment. For example:

```yaml
################################
# PLEX
################################

    - name: Prepare plex token
      when: inventory_hostname == 'localhost'
      ansible.builtin.include_role:
        name: plex
        apply:
          tags: plex
      tags: plex

    - name: Prepare plex stack
      when: inventory_hostname == 'plex'
      ansible.builtin.include_role:
        name: plex2
        apply:
          tags: plex
      tags: plex

    - name: Deploy plex stack
      when: inventory_hostname == 'localhost'
      ansible.builtin.include_role:
        name: plex3
        apply:
          tags: plex
      tags: plex
```

***

### Tags

***

The tags in my `playbook.yml` are what I use to determine what job I want to run, what docker services I want to deploy. There are two main types of tags that need to be included, as seen here:

```yaml
    - name: Deploy plex stack
      when: inventory_hostname == 'localhost'
      ansible.builtin.include_role:
        name: plex3
        apply:
          tags: plex
      tags: plex
```

In this example, the `tags: arrs` calls the role during the play, and the `apply.tags: arrs` applies the tag to the tasks within the role, ensuring these tasks run when the tag is used. 

You can apply as many tags as you require for yourset up, and can also apply `tags: always` for those tasks that always need to run, like in my pre_tasks section.

And as seen in the introduction, you simply include your desired tags in your playbook run command:

```cli

ansible-playbook play.yml -i hosts.ini --ask-become-pass --ask-vault-pass --tag plex

```


***

## Handlers

***

Handlers are tasks that will only run when notified, will run at the end of each play, and will only run once no matter how many times they're notified.

A recent example of where I have made use of Handlers was with the setting up of mounts for autofs:

```yaml

- name: Append desired local mount path to autofs auto.master file
  ansible.builtin.lineinfile:
    dest: '{{ autofs_auto_mast_file }}'
    line: '{{ autofs_local_path }} {{ autofs_nfsdb_file }} --timeout=0 --browse'
    state: present
    create: true
    owner: '{{ puid }}'
    group: '{{ pgid }}'
    mode: '0644'
  notify:
    - redo mounts

- name: Add media mount to auto.nfsdb file
  ansible.builtin.lineinfile:
    dest: '{{ autofs_nfsdb_file }}'
    line: '{{ autofs_media_dir }} {{ autofs_media_opt }} {{ autofs_media_address }}:{{ autofs_media_path }}'
    state: present
    create: true
    owner: '{{ puid }}'
    group: '{{ pgid }}'
    mode: '0644'
  notify:
    - redo mounts

```

In the above these tasks notify the following handlers that are listening for 'redo mounts':


```yaml

################################
# HANDLERS
################################

  ## Examples of some handlers I used before moving from autofs to mergerfs

  handlers:
    - name: Stop autofs service
      ansible.builtin.service:
        name: autofs
        state: stopped
      listen: redo mounts

    - name: Unmount autofs mounts
      ansible.builtin.command: umount -a -t autofs
      listen: redo mounts

    - name: Start autofs service
      ansible.builtin.service:
        name: autofs
        state: started
      listen: redo mounts

```

Depending on how many handlers you have, you can either define them directly in your playbook, have them in a handlers folder, or even define them within individual roles.
