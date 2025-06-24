---
draft: false
genres:
- ansible
tags:
- common-tasks
- templates
- ansible
title: Templates
---

## Common Tasks - Templates

One of the most important tasks and one of the primary reasons I'm using Ansible is to set-up the various configs for my VMs and the services residing on them. For this, I typically template a basic config file from a role directory into the desired system or service folder:


```yaml
- name: Template file
  ansible.builtin.template:
    src: '{{ template_source }}'
    dest: '{{ template_destination }}'
    force: '{{ template_force }}'
    owner: '{{ template_owner }}'
    group: '{{ template_group }}'
    mode: '{{ template_mode }}'

- name: Wait for file to be created
  ansible.builtin.wait_for:
    path: '{{ template_destination }}'
    state: present
```

**Include Tasks:

```yaml
- name: Conduct template tasks
  ansible.builtin.include_tasks: '/ansible/resources/template.yml'
  vars:
    template_location: '{{ item.template }}'
    file_location: '{{ item.file }}'
  loop:
    - { template: '{{ role_path }}/templates/configs/autobrr_config.toml.j2', file: '{{ autobrr_location }}/config.toml' }
    - { template: '{{ role_path }}/templates/configs/doplarr_config.edn.j2', file: '{{ doplarr_location }}/config.edn' }

```

**Example config (autobrr_config.toml.j2):

```yaml
# config.toml

host = '0.0.0.0'
port = '{{ autobrr_ports_cont }}'
logPath = '{{ autobrr_logs_path }}'
logLevel = '{{ autobrr_logs_level }}'
logMaxSize = '{{ autobrr_logs_max_size }}'
logMaxBackups = '{{ autobrr_logs_max_backups }}'
checkForUpdates = true
sessionSecret = '{{ autobrr_session_secret.stdout }}'
```

**After templating:

```yaml
# config.toml

host = '0.0.0.0'
port = '7474'
logPath = '/log/autobrr.log'
logLevel = 'DEBUG'
logMaxSize = '50'
logMaxBackups = '3'
checkForUpdates = true
sessionSecret = 'SomeSecret'
```