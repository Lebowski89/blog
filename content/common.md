---
draft: false
genres:
- ansible
tags:
- common-tasks
- ansible
- cloudflare
- traefik
menus: common
weight: 1
title: Handling common tasks within Ansible
---

***

## Introduction

***

When you begin constructing your Ansible tasks - roles - collections, you quickly realise that there are many tasks that you will use repeatedly with little variance. For example, if you're using Ansible to help set-up and deploy your docker containers, you'll have common tasks that create directories, configs, DNS and Traefik labels, among many others. If you're like me, I would simply copy and paste these tasks. While Ansible will run through and automate these repeated tasks all the same, the more tasks you have to repeat in a play, the more bloated and unreadable your playbook becomes and the more maintenance upkeep is required.

To handle commonly repeated tasks, I've found it is effective to make a common tasks, in which you include during plays as required. In these tasks are generic variables that are looped over and replaced with relevant variables as required.

The rest of this document is dedicated to describing the common tasks I use.

### Key points
   - The common tasks are included in a resources directory in the Ansible directory
   - Each task has generic variables that will be replaced with relevant ones during the play when the role is included using the `ansible.builtin.include_tasks` module
   - Loops and iterating over hashes are key to reducing the number of required tasks.


***

## Cloudflare DNS

***

One integral time-saving task when spinning up docker services is to automate DNS records.

### Common Task

```yaml
- name: Remove existing A DNS record
  when: cloudflare_remove_existing == 'true'
  community.general.cloudflare_dns:
    api_token: '{{ cloudflare_api }}'
    zone: '{{ cloudflare_domain }}'
    state: absent
    type: '{{ cloudflare_type }}'
    record: '{{ cloudflare_record }}'

- name: Perform Cloudflare DNS tasks
  block:
    - name: Add DNS record
      community.general.cloudflare_dns:
        api_token: '{{ cloudflare_api }}'
        zone: '{{ cloudflare_domain }}'
        state: present
        solo: '{{ cloudflare_solo }}'
        proxied: '{{ cloudflare_proxy }}'
        type: '{{ cloudflare_type }}'
        value: '{{ cloudflare_value }}'
        record: '{{ cloudflare_record }}'
      register: cloudflare_record_creation_status

    - name: Tasks on success
      when: cloudflare_record_creation_status is succeeded
      block:
        - name: Set 'dns_record_print' variable
          ansible.builtin.set_fact:
            cloudflare_record_print: '{{ (cloudflare_record == cloudflare_domain) | ternary(cloudflare_domain, cloudflare_record + "." + cloudflare_domain) }}'

        - name: Display DNS record creation status
          ansible.builtin.debug:
            msg: 'DNS A Record for "{{ cloudflare_record_print }}" set to "{{ cloudflare_value }}" was added. Proxy: {{ cloudflare_proxy }}'

```

These will add/remove DNS records and display the DNS record on success.

### include_task

```yaml
- name: Add Cloudflare DNS records
  ansible.builtin.include_tasks: '/ansible/resources/cloudflare.yml'
  vars:
    cloudflare_domain: '{{ local_domain }}'
    cloudflare_record: '{{ obsidian_name }}'
    cloudflare_type: 'A'
    cloudflare_value: '{{ ipify_public_ip }}'
    cloudflare_proxy: 'false'
    cloudflare_solo: 'true'
    cloudflare_remove_existing: 'true'
```

In the above example, variables from the common task are replaced with relevant values required to create a DNS record for my Obsidian container running on my local server. You can create loops to handle as many services as you require:

```yaml
- name: Add 'GitHub Pages' DNS records
  ansible.builtin.include_tasks: '/ansible/resources/cloudflare.yml'
  vars:
    cloudflare_domain: '{{ github_pages_domain }}'
    cloudflare_record: '@'
    cloudflare_type: '{{ item.type }}'
    cloudflare_value: '{{ item.value }}'
    cloudflare_proxy: 'false'
    cloudflare_solo: 'false'
    cloudflare_remove_existing: 'false'
  loop:
    - { type: 'A', value: '185.199.108.153' }
    - { type: 'A', value: '185.199.109.153' }
    - { type: 'A', value: '185.199.110.153' }
    - { type: 'A', value: '185.199.111.153' }
    - { type: 'AAAA', value: '2606:50c0:8000::153' }
    - { type: 'AAAA', value: '2606:50c0:8001::153' }
    - { type: 'AAAA', value: '2606:50c0:8002::153' }
    - { type: 'AAAA', value: '2606:50c0:8003::153' }
```

With each loop the variables are replaced without interfering with others.

***

## File Copy

***

A simple task to copy files from one directory to another. I typically use this to copy files that docker services require from their role folder to the services appdata directory (though, I prefer to template files where and when possible).

### Common Task

```yaml
- name: Check if file exists
  ansible.builtin.stat:
    path: '{{ copy_destination }}'
  register: file_copy_stat

- name: Copy file
  when: not file_copy_stat.stat.exists
  ansible.builtin.copy:
    src: '{{ copy_source }}'
    dest: '{{ copy_destination }}'
    force: '{{ copy_force }}'
    owner: '{{ copy_owner }}'
    group: '{{ copy_pgid }}'
    mode: '{{ copy_mode }}'

- name: Wait for file to be created
  ansible.builtin.wait_for:
    path: '{{ copy_destination }}'
    state: present
```

### include_task

```yaml
- name: Copy Hugo Terminal Themes files
  ansible.builtin.include_tasks: '/ansible/resources/copy.yml'
  vars:
    copy_source: '{{ item.source }}'
    copy_destination: '{{ item.destination }}'
    copy_force: 'true'
    copy_owner: '{{ puid }}'
    copy_group: '{{ pgid }}'
    copy_mode: '0644'
  loop:
    - { source: '{{ role_path }}/files/favicon.png', destination: '/{{ hugo_site_name }}/static/favicon.png' }
    - { source: '{{ role_path }}/files/og-image.png', destination: '/{{ hugo_site_name }}/static/og-image.png' }
    - { source: '{{ role_path }}/files/terminal.css', destination: '/{{ hugo_site_name }}/static/terminal.css' }
```

In the above example, I copy the files required for this blogs theme.

***

## Templates

***

One of the most important tasks, and one of the primary reasons I'm using Ansible, is to set-up the various configs for my VMs and the services in them. For this, I typically template a config file from a role directory into a desired folder.

### Common Task

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

### include_task

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

**Example config (autobrr_config.toml.j2):**

```toml
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

**After templating:**

```toml
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

***

## Traefik Labels

***

Traefik is my reverse-proxy of choice for my docker services, with much of the config taking place in Traefik docker labels. Previously, I would define the Traefik labels for each service individually in a role defaults file. Depending on the service, I have http, https, api and theme-park labels. It can get quite extensive and much of the labels are the same across services with only variables such as the router name and port differing. 

To reduce the bloat, I make use of:
   1. A labels template contained in my group_vars
   2. A common task file to determine required labels
   3. Include_tasks task with relevant variables.

### Group_Vars

```yaml
traefik_labels_core:
  - traefik.enable: "true"
  - traefik.docker.network: "{{ router_network }}"

traefik_labels_http:
  - '{ "traefik.http.routers.{{ router_name }}.entrypoints": "{{ router_entrypoint }}" }'
  - '{ "traefik.http.routers.{{ router_name }}.rule": "{{ router_rule }}" }'
  - '{ "traefik.http.routers.{{ router_name }}.middlewares": "{{ router_http_middlewares }}" }'
  - '{ "traefik.http.routers.{{ router_name }}.priority": "20" }'
  - '{ "traefik.http.routers.{{ router_name }}.service": "{{ router_name }}-svc" }'
  - '{ "traefik.http.services.{{ router_name }}-svc.loadbalancer.server.port": "{{ router_port }}" }'

traefik_labels_https:
  - '{ "traefik.http.routers.{{ router_name }}-secure.entrypoints": "{{ router_secure_entrypoint }}" }'
  - '{ "traefik.http.routers.{{ router_name }}-secure.rule": "{{ router_rule }}" }'
  - '{ "traefik.http.routers.{{ router_name }}-secure.middlewares": "{{ router_https_middlewares }}" }'
  - '{ "traefik.http.routers.{{ router_name }}-secure.priority": "20" }'
  - '{ "traefik.http.routers.{{ router_name }}-secure.service": "{{ router_name }}-secure-svc" }'
  - '{ "traefik.http.routers.{{ router_name }}-secure.tls.certresolver": "{{ router_tls_certresolver }}" }'
  - '{ "traefik.http.routers.{{ router_name }}-secure.tls.options": "{{ router_tls_options }}" }'
  - '{ "traefik.http.services.{{ router_name }}-secure-svc.loadbalancer.server.port": "{{ router_port }}" }'

traefik_labels_http_api:
  - '{ "traefik.http.routers.{{ router_name }}-api.entrypoints": "{{ router_entrypoint }}" }'
  - '{ "traefik.http.routers.{{ router_name }}-api.rule": "{{ router_api_rule }}" }'
  - '{ "traefik.http.routers.{{ router_name }}-api.middlewares": "{{ traefik_http_middlewares }}" }'
  - '{ "traefik.http.routers.{{ router_name }}-api.priority": "30" }'
  - '{ "traefik.http.routers.{{ router_name }}-api.service": "{{ router_name }}-svc" }'

traefik_labels_https_api:
  - '{ "traefik.http.routers.{{ router_name }}-api-secure.entrypoints": "{{ router_secure_entrypoint }}" }'
  - '{ "traefik.http.routers.{{ router_name }}-api-secure.rule": "{{ router_api_rule }}" }'
  - '{ "traefik.http.routers.{{ router_name }}-api-secure.middlewares": "{{ traefik_https_middlewares }}" }'
  - '{ "traefik.http.routers.{{ router_name }}-api-secure.priority": "30" }'
  - '{ "traefik.http.routers.{{ router_name }}-api-secure.service": "{{ router_name }}-secure-svc" }'
  - '{ "traefik.http.routers.{{ router_name }}-api-secure.tls.certresolver": "{{ router_tls_certresolver }}" }'
  - '{ "traefik.http.routers.{{ router_name }}-api-secure.tls.options": "{{ router_tls_options }}" }'

traefik_labels_themepark:
  - '{ "traefik.http.middlewares.themepark-{{ router_name }}.plugin.themepark.app": "{{ router_themepark_app }}" }'
  - '{ "traefik.http.middlewares.themepark-{{ router_name }}.plugin.themepark.theme": "{{ router_themepark_theme }}" }'
```

### Common Task

```yaml
################################
# CHECKS
################################

- name: Set api fact
  ansible.builtin.set_fact:
    router_api_is_enabled: '{{ (router_api is defined) and
                               (router_api is not none) and
                               (router_api | trim | length > 0) }}'

- name: Set themepark fact
  ansible.builtin.set_fact:
    router_themepark_is_enabled: '{{ (router_themepark_app is defined) and
                                     (router_themepark_app is not none) and
                                     (router_themepark_app | trim | length > 0) }}'

################################
# RULES
################################

- name: Set router rule
  ansible.builtin.set_fact:
    router_rule: '{{ "Host(`" + router_name + "." + router_domain + "`)" }}'

- name: Set router api rule
  when: router_api_is_enabled
  ansible.builtin.set_fact:
    router_api_rule: '{{ "Host(`" + router_name + "." + router_domain + "`) && (" + router_api + ")" }}'

################################
# LABELS
################################

- name: Set basic traefik labels
  when:
    - not router_themepark_is_enabled
    - not router_api_is_enabled
  ansible.builtin.set_fact:
    '{{ router_variable }}': '{{ traefik_labels_core
                                 | combine(traefik_labels_http)
                                 | combine(traefik_labels_https) }}'

- name: Set api traefik labels
  when:
    - not router_themepark_is_enabled
    - router_api_is_enabled
  ansible.builtin.set_fact:
    '{{ router_variable }}': '{{ traefik_labels_core
                                 | combine(traefik_labels_http)
                                 | combine(traefik_labels_https)
                                 | combine(traefik_labels_http_api) 
                                 | combine(traefik_labels_https_api) }}'

- name: Set api-only traefik labels
  when:
    - not router_themepark_is_enabled
    - router_api_is_enabled
    - router_variable is search('_api_labels')
  ansible.builtin.set_fact:
    '{{ router_variable }}': '{{ traefik_labels_core
                                 | combine(traefik_labels_http_api) 
                                 | combine(traefik_labels_https_api) }}'

- name: Set themepark traefik labels
  when:
    - router_themepark_is_enabled
    - not router_api_is_enabled
  ansible.builtin.set_fact:
    '{{ router_variable }}': '{{ traefik_labels_core
                                 | combine(traefik_labels_http)
                                 | combine(traefik_labels_https)
                                 | combine(traefik_labels_themepark) }}'

- name: Set full traefik labels
  when:
    - router_themepark_is_enabled
    - router_api_is_enabled
  ansible.builtin.set_fact:
    '{{ router_variable }}': '{{ traefik_labels_core
                                 | combine(traefik_labels_http)
                                 | combine(traefik_labels_https)
                                 | combine(traefik_labels_http_api) 
                                 | combine(traefik_labels_https_api)
                                 | combine(traefik_labels_themepark) }}'
```

The above task file combines the relevant labels into a single variable that is provided to the docker compose file.

### include_task

```yaml
- name: Set traefik Labels
  ansible.builtin.include_tasks: /ansible/resources/labels.yml
  vars:
    router_variable: '{{ item.var }}'
    router_network: '{{ network_overlay }}'
    router_name: '{{ item.name }}'
    router_port: '{{ item.port }}'
    router_api: '{{ item.api }}'
    router_domain: '{{ local_domain }}'
    router_entrypoint: 'http'  ## web if saltbox
    router_secure_entrypoint: 'https'  ## websecure if saltbox
    router_tls_certresolver: 'dns-cloudflare'  ## cfdns if saltbox
    router_tls_options: 'tls-opts@file'  ## securetls@file if saltbox
    router_themepark_app: '{{ item.tp_app }}'
    router_themepark_theme: 'hotpink'
    router_http_middlewares: '{{ traefik_http_middlewares + item.sso + item.tp }}'
    router_https_middlewares: '{{ traefik_https_middlewares + item.sso + item.tp }}'
  loop:
    ## bazarr
    - { var: 'bazarr_labels', 
        name: '{{ bazarr_name }}', 
        port: '{{ bazarr_ports_cont }}', 
        api: 'PathPrefix(`/api`)', 
        sso: ',authelia@swarm', 
        tp: ',themepark-bazarr', 
        tp_app: 'bazarr' }
    ## lidarr
    - { var: 'lidarr_labels', 
        name: '{{ lidarr_name }}', 
        port: '{{ lidarr_ports_cont }}', 
        api: 'PathPrefix(`/api`) || PathPrefix(`/feed`) || PathPrefix(`/ping`)', 
        sso: ',authelia@swarm', 
        tp: ',themepark-lidarr', 
        tp_app: 'lidarr' }
```

In the above example, both services receive full Traefik labels, including being protected by my SSO provider (Authelia), API labels and Theme-Park. Simply leaving the API and/or Themepark vars empty will remove these labels, i.e:

```yaml
- name: Set Obsidian traefik Labels
  ansible.builtin.include_tasks: /ansible/resources/labels.yml
  vars:
    router_variable: 'obsidian_labels'
    router_network: '{{ network_overlay }}'
    router_name: '{{ obsidian_name }}'
    router_port: '{{ obsidian_ports_http_cont }}'
    router_api: ''
    router_domain: '{{ local_domain }}'
    router_entrypoint: 'http'  ## web if saltbox
    router_secure_entrypoint: 'https'  ## websecure if saltbox
    router_tls_certresolver: 'dns-cloudflare'  ## cfdns if saltbox
    router_tls_options: 'tls-opts@file'  ## securetls@file if saltbox
    router_themepark_app: ''
    router_themepark_theme: 'hotpink'
    router_http_middlewares: '{{ traefik_http_middlewares + ",authelia@swarm" }}'
    router_https_middlewares: '{{ traefik_https_middlewares + ",authelia@swarm" }}'
```

Moving Traefik labels to this eliminated unnecessary defaults files.


***

## Postgres Database

***

I use postgres as a database for a variety of docker services, across multiple roles. As such, I use the `postgresql.postgresql_ping` and `postgresql.postgresql_db` modules to ping for an existing database and to create a one if none exists. These tasks require an existing postgres install that is up and running (I deploy postgres via docker swarm), and the host (machine ip)/user/port/password of the running instance.


### Common Task

```yaml

- name: Ping for existing database
  community.postgresql.postgresql_ping:
    login_host: '{{ pvr_machine }}'
    login_user: '{{ postgres_username }}'
    login_password: '{{ postgres_password }}'
    port: '{{ postgres_ports_host }}'
    login_db: '{{ postgres_database }}'
  register: postgres_db_exists

- name: Create postgres database
  when: not postgres_db_exists == true
  community.postgresql.postgresql_db:
    login_host: '{{ pvr_machine }}'
    login_user: '{{ postgres_username }}'
    login_password: '{{ postgres_password }}'
    port: '{{ postgres_ports_host }}'
    name: '{{ postgres_database }}'
    state: present

```

### include_task

```yaml

- name: Conduct Postgres DB tasks
  ansible.builtin.include_tasks: /ansible/common/postgres.yml
  vars:
    postgres_database: '{{ item }}'
  loop:
    - 'bazarr'
    - 'lidarr-main'
    - 'lidarr-log'
    - 'prowlarr-main'
    - 'prowlarr-log'
    - 'radarr-main'
    - 'radarr-log'
    - 'radarr-4k-main'
    - 'radarr-4k-log'
    - 'readarr-main'
    - 'readarr-log'
    - 'readarr-cache'
    - 'sonarr-main'
    - 'sonarr-log'
    - 'sonarr-4k-main'
    - 'sonarr-4k-log'
    - 'whisparr-main'
    - 'whisparr-log'
    - 'whisparr-main-v3'
    - 'whisparr-log-v3'

```

Above, the only thing that changes for the include task is the name of the database. I'm only working with one postgres instance, so that's all I need.

<script data-name="BMC-Widget" data-cfasync="false" src="https://cdnjs.buymeacoffee.com/1.0.0/widget.prod.min.js" data-id="lebowski89" data-description="Support me on Buy me a coffee!" data-message="Coffee" data-color="#5F7FFF" data-position="Right" data-x_margin="18" data-y_margin="18"></script>