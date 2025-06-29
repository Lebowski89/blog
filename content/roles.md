---
draft: false
genres:
- ansible
tags:
- roles
- ansible
menus: roles
weight: 1
title: Structuring your Ansible Roles
---

***

## Introduction

***

Ansible roles contain the majority of my tasks that automate the setup and deployment of my Docker containers and Swarm services. During my time with Ansible, my roles have changed a lot, both content and structure wise. 


***

## Group_Vars

***

For my set up, group_vars, is structured like the following:

```cli
/ansible/group_vars
└── all
    ├── docker.yml
    ├── roles
    │   ├── arrs.yml
    │   ├── blog.yml
    │   ├── comps.yml
    │   ├── dns.yml
    │   ├── gitea.yml
    │   ├── mariadb.yml
    │   ├── metrics.yml
    │   ├── plex.yml
    │   ├── postgres.yml
    │   ├── proxy.yml
    │   ├── torrents.yml
    │   ├── unifi.yml
    │   ├── usenet.yml
    │   ├── utilities.yml
    │   ├── vpn.yml
    │   └── wordpress.yml
    ├── traefik.yml
    └── vault.yml
```

1. The `docker.yml` file contains general docker variables, such as puid/pgid and docker network information 
2. The `traefik.yml` file contains traefik middleware and a traefik labels template variables (used by the traefik labels common task)
3. The `vault.yml` file contains all sensitive variables, including api/keys/domains/passwords/emails and any other sensitive information.

The remaining variables are contained in the `group_vars/all/roles` folder, with one group_vars file per role. These contain all the relevant information required to get the roles services up and running, for example:

```yaml
radarr_name: 'radarr'
radarr_image_repo: 'ghcr.io/hotio/radarr'
radarr_image_tag: 'latest'
radarr_ports_host: '7878'
radarr_ports_cont: '7878'
radarr_location: '/opt/{{ radarr_name }}'
radarr_root_folder: '/media/movies'
radarr_quality_profile: 'Remux + WEB 1080p'
```

In the above example, the name/ports/location are required by the arrs role to deploy Radarr.

The reason these are defined in group_vars rather than within each role is because these variables may be required by not just one role but multiple roles, for example:
  - Role 1 - Prepares and deploys Radarr
  - Role 2 - Prepares and deploys Doplarr and Jellyseerr
  - Role 3 - Prepares and deploys Radarr Prometheus exporter

In the above example, all three roles will require Radarr's service name, port, location and api (in group_vars/vault). Rather than defining these multiple times, it is defined once in group_vars. Previously, I would only include these variables that were required by multiple roles in group_vars and the rest within the role/defaults folder, but this led to a mish-mash of defaults in different locations, so I simply moved all to group_vars. 

***

## Defaults

***

Each role contains 'defaults' variables that are specific to the services being deployed. For roles with single/few services, defaults are defined in a single main.yml file (role/defaults/main.yml), but most services will have their own defaults file (role/defaults/main/someservice.yml). Common defaults include names, ports, locations, traefik labels, and any other variable not in group_vars or created during the play. With a defaults/main folder, you can have as many defaults files as you like.

**Example:**

```yaml
## metrics/defaults/main/exporters.yml

################################
# BASICS
################################

bazarr_exporter_defaults_name: '{{ bazarr_name }}-exporter'
bazarr_exporter_defaults_image_repo: 'ghcr.io/onedr0p/exportarr'
bazarr_exporter_defaults_image_tag: 'latest'

lidarr_exporter_defaults_name: '{{ lidarr_name }}-exporter'
lidarr_exporter_defaults_image_repo: 'ghcr.io/onedr0p/exportarr'
lidarr_exporter_defaults_image_tag: 'latest'

plex_exporter_defaults_name: 'plex-exporter'
plex_exporter_defaults_image_repo: 'ghcr.io/axsuul/plex-media-server-exporter'
plex_exporter_defaults_image_tag: 'latest'

## Note: bazarr_name and lidarr_name are defined in group_vars (see above).
```


***

## Files

***

When I want to copy a file into a service directory, I include them in the role/files folder. Generally, I prefer templates and/or creating files during the play. But there are some cases where I copy. One example includes the various themepark scripts (i.e, [Sonarr](https://github.com/themepark-dev/theme.park/blob/master/docker-mods/sonarr/root/etc/cont-init.d/98-themepark)), which are included as role/files/98-themepark-someapp and then copied using the following role tasks:

  ```
################################
# THEME
################################

- name: Check if theme-park script exists
  ansible.builtin.stat:
    path: '{{ sonarr_defaults_location }}/98-themepark-sonarr'
  register: sonarr_theme_script

- name: Copy theme-park script
  when: not sonarr_theme_script.stat.exists
  ansible.builtin.copy:
    src: '{{ role_path }}/files/98-themepark-sonarr'
    dest: '{{ sonarr_defaults_location }}/98-themepark-sonarr'
    owner: '{{ puid }}'
    group: '{{ pgid }}'
    mode: '0775'
```

***

## Tasks

***

Removes existing stacks, retrieves common vault variables, includes sub_tasks, and then deploys the stack.

**Example:**

```
################################
# CLEAN UP
################################

- name: Remove arrs stack
  community.docker.docker_stack:
    name: arrs
    state: absent

- name: Remove arrs-stack file
  ansible.builtin.file:
    path: /opt/compose/arrs-stack.yml
    state: absent

################################
# VAULT
################################

- name: Load common vault variables
  ansible.builtin.set_fact:
    cloudflare_api: '{{ lookup("ini", "cloudflare_api section=" + "traefik" + " file=" + "/ansible/vault.ini") }}'
    local_domain: '{{ lookup("ini", "local_domain section=" + "traefik" + " file=" + "/ansible/vault.ini") }}'
    postgres_username: '{{ lookup("ini", "username section=" + "postgres" + " file=" + "/ansible/vault.ini") }}'
    postgres_password: '{{ lookup("ini", "password section=" + "postgres" + " file=" + "/ansible/vault.ini") }}'

################################
# SUB-TASKS
################################

- name: Include sub-tasks tasks
  ansible.builtin.include_tasks: sub_tasks/{{ item }}.yml
  loop:
    - 'rclone'  ## rclone runs first to create the torrents volume
    - 'lidarr'
    - 'prowlarr'
    - 'radarr'
    - 'radarr4k'
    - 'readarr'
    - 'sonarr'
    - 'sonarr4k'
    - 'whisparr'
    - 'bazarr'  ## bazarr relies on arrs API variables - run last

################################
# DEPLOY
################################

- name: Import arrs-stack file
  ansible.builtin.template:
    src: '{{ role_path }}/templates/arrs-stack.yml.j2'
    dest: /opt/compose/arrs-stack.yml
    force: true
    owner: '{{ puid }}'
    group: '{{ pgid }}'
    mode: '0664'

- name: Deploy arrs stack
  community.docker.docker_stack:
    state: present
    name: arrs
    compose:
      - /opt/compose/arrs-stack.yml
```

***

### Sub-tasks

***

Sub_tasks prepare a docker swarm service to run, including:
  * Creating directories
  * Creating and preparing config files
  * Creating databases and adding database info to the config files 
  * Retrieving vault variables or generating keys/api/passwords during the play
  * Preparing docker secrets
  * Creating cloudflare A/CNAME DNS records
  and so on...

**Example:**

```
## companions/tasks/sub_tasks/ombi.yml

################################
# DIRECTORY
################################

- name: Create appdata directories
  ansible.builtin.file:
    path: '{{ item }}'
    state: directory
    owner: '{{ puid }}'
    group: '{{ pgid }}'
    mode: '0755'
  loop:
    - '{{ ombi_defaults_location }}'
    - '{{ ombi_defaults_location }}/config'

################################
# POSTGRES
################################

- name: Ping for existing database
  community.postgresql.postgresql_ping:
    login_host: '{{ pvr_machine }}'
    login_user: '{{ postgres_username }}'
    login_password: '{{ postgres_password }}'
    port: '{{ postgres_ports_host }}'
    login_db: 'Ombi'
  register: ombi_postgres_db

- name: Create postgres database
  when: not ombi_postgres_db == true
  community.postgresql.postgresql_db:
    login_host: '{{ pvr_machine }}'
    login_user: '{{ postgres_username }}'
    login_password: '{{ postgres_password }}'
    port: '{{ postgres_ports_host }}'
    name: 'Ombi'
    state: present

- name: Import ombi database.json
  ansible.builtin.template:
    src: '{{ role_path }}/templates/configs/ombi_database.json.j2'
    dest: '{{ ombi_defaults_location }}/config/database.json'
    force: true
    owner: '{{ puid }}'
    group: '{{ pgid }}'
    mode: '0664'

- name: Remove sqlite database files
  ansible.builtin.file:
    path: '{{ ombi_defaults_location }}/{{ item }}'
    state: absent
  loop:
    - Ombi.db
    - OmbiExternal.db
    - OmbiSettings.db

################################
# CLOUDFLARE
################################

- name: Perform Cloudflare DNS tasks
  block:
    - name: Remove existing CNAME DNS record
      community.general.cloudflare_dns:
        api_token: '{{ cloudflare_api }}'
        zone: '{{ local_domain }}'
        state: 'absent'
        type: 'CNAME'
        record: '{{ ombi_name }}'

    - name: Remove existing A DNS record
      community.general.cloudflare_dns:
        api_token: '{{ cloudflare_api }}'
        zone: '{{ local_domain }}'
        state: 'absent'
        type: 'A'
        record: '{{ ombi_name }}'

    - name: Add DNS record
      community.general.cloudflare_dns:
        api_token: '{{ cloudflare_api }}'
        zone: '{{ local_domain }}'
        state: 'present'
        solo: true
        proxied: '{{ cloudflare_proxy }}'
        type: '{{ cloudflare_record }}'
        value: '{{ ipify_public_ip }}'
        record: '{{ ombi_name }}'
      register: ombi_cloudflare_record_creation_status

    - name: Tasks on success
      when: ombi_cloudflare_record_creation_status is succeeded
      block:
        - name: Set 'dns_record_print' variable
          ansible.builtin.set_fact:
            ombi_cloudflare_record_print: '{{ (ombi_name == local_domain) | ternary(local_domain, ombi_name + "." + local_domain) }}'

        - name: Display DNS record creation status
          ansible.builtin.debug:
            msg: 'DNS A Record for "{{ ombi_cloudflare_record_print }}" set to "{{ ipify_public_ip }}" was added. Proxy: {{ cloudflare_proxy }}'
```

***

## Templates

***



***

### Configs

***



Contains nearly all my service config files, ready to be called upon during play. I find templating more powerful than simply copying, as a change in a variable's value will apply to all templates referencing it - no need to edit every config file individually. 

**Example:**

```
## Doplarr's config (config.edn.j2)

{
 :radarr/url "http://{{ radarr_name }}:{{ radarr_ports_cont }}"
 :radarr/api "{{ radarr_api }}"
 :radarr/quality-profile "{{ radarr_quality_profile }}"
 :radarr/rootfolder "{{ radarr_root_folder }}"
 :sonarr/url "http://{{ sonarr_name }}:{{ sonarr_ports_cont }}"
 :sonarr/api "{{ sonarr_api }}"
 :sonarr/quality-profile "{{ sonarr_quality_profile }}"
 :sonarr/rootfolder "{{ sonarr_root_folder }}"
 :sonarr/season-folders true
 :partial-seasons true
 :discord/token "{{ doplarr_token }}"
 :discord/max-results 25
 :discord/requested-msg-style :plain
 :log-level :trace
 }
 ```
 
**Which is then called with the following tasks:**

```
################################
# CONFIG
################################

- name: Check if doplarr config.edn exists
  ansible.builtin.stat:
    path: '{{ doplarr_defaults_location }}/config.edn'
  register: doplarr_config_edn

- name: Load doplarr vault variable
  when: not doplarr_config_edn.stat.exists
  ansible.builtin.set_fact:
    doplarr_token: '{{ lookup("ini", "doplarr_token section=" + "companions" + " file=" + "/ansible/vault.ini") }}'

- name: Create doplarr config.edn file
  when: not doplarr_config_edn.stat.exists
  block:
    - name: Import config file
      ansible.builtin.template:
        src: '{{ role_path }}/templates/configs/doplarr_config.edn.j2'
        dest: '{{ doplarr_defaults_location }}/config.edn'
        force: true
        owner: '{{ puid }}'
        group: '{{ pgid }}'
        mode: '0664'

    - name: Wait for 'config.edn' to be created
      ansible.builtin.wait_for:
        path: '{{ doplarr_defaults_location }}/config.edn'
        state: present
```

***

### Compose

***

Aside from the configs, the docker compose file is also templated, containing relevant (to the role) and related docker services.

**Example:**

```
version: '3.9'

services:
  {{ adguard_defaults_name }}:
    image: {{ adguard_defaults_image_repo }}:{{ adguard_defaults_image_tag }}
    networks:
      - {{ network_overlay }}
    environment:
      PUID: '{{ puid }}'
      PGID: '{{ pgid }}'
      TZ: '{{ timezone }}'
    ports:
      - target: {{ adguard_defaults_ports_cont }}
        published: {{ adguard_defaults_ports_host }}
        protocol: tcp
        mode: ingress
      - target: {{ adguard_defaults_ports_dns_cont }}
        published: {{ adguard_defaults_ports_dns_host }}
        protocol: tcp
        mode: ingress
      - target: {{ adguard_defaults_ports_dns_cont }}
        published: {{ adguard_defaults_ports_dns_host }}
        protocol: udp
        mode: ingress
    volumes:
      - type: bind
        source: {{ adguard_defaults_location }}/work
        target: /opt/adguardhome/work
      - type: bind
        source: {{ adguard_defaults_location }}/conf
        target: /opt/adguardhome/conf
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.labels.ansible_host == pvr]
      labels: {{ adguard_defaults_labels }}

networks:
  {{ network_overlay }}:
    external: true
```

With the compose files, you can either create and define networks, volumes, secrets within the file, or you can let ansible handle it and then use 'external: true'. I prefer the latter and use various docker ansible modules to automate their creation:

  **Networks:**
```
- name: Conduct overlay network tasks
  when: inventory_hostname == 'localhost'
  block:
    - name: Register overlay network
      community.docker.docker_network_info:
        name: '{{ network_overlay }}'
      register: network_overlay_result

    - name: Create overlay network
      when: not network_overlay_result.exists
      community.docker.docker_network:
        name: '{{ network_overlay }}'
        driver: '{{ network_overlay_driver }}'
        attachable: true 
        ipam_config:
          - subnet: '{{ network_overlay_subnet }}'
```

  **Volumes:**
```
- name: Create media volume
  community.docker.docker_volume:
    volume_name: '{{ media_volume }}'
    state: present
    recreate: options-changed
    driver: local
    driver_options:
      type: nfs
      o: 'addr={{ media_volume_address }},rw,nfsvers=4.2'
      device: '{{ media_volume_device }}'
```

  **Secrets:**
```
- name: Prepare docker secrets
  block:
    - name: Unifi db user secret
      community.docker.docker_secret:
        name: unifi_user_secret
        data: '{{ unifi_db_user }}'
        state: present

    - name: Unifi db password secret
      community.docker.docker_secret:
        name: unifi_pass_secret
        data: '{{ unifi_db_pass }}'
        state: present

    - name: Unifi db root password secret
      community.docker.docker_secret:
        name: unifi_root_pass_secret
        data: '{{ unifi_db_root_pass }}'
        state: present
 ```