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

Above, the name/ports/location are required by the arrs role to deploy Radarr.

The reason these are defined in group_vars rather than within each role is because these may be required by not just one role but multiple, for example:
  - Role 1 - Prepares and deploys Radarr
  - Role 2 - Prepares and deploys Doplarr and Jellyseerr
  - Role 3 - Prepares and deploys Radarr Prometheus exporter

In the above example, all three roles will require Radarr's service name, port, location and api. Rather than defining these multiple times, they're defined once in group_vars. 

Remember: The benefit of defaults variables, aside from sharing information between roles, is to have the ability to quickly change their value and have it represented throughout all roles/tasks that use it.

***

### Role/Defaults

***

Previously, I would include defaults required by multiple roles in group_vars and the remainder in role defaults, but this led to a mish-mash of defaults in different locations. With the adoption of common tasks to handle things, such as traefik labels, I was able to minimise the amount of defaults and moved all to group_vars. One thing I did like to do when using role defaults was to include 'defaults' in the variable name:

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

You also might like to have role defaults that are used throughout the role, with these defaults pointing to group_vars. I simply prefer to have as few variables as possible.


***

## Role/Files

***

When I want to copy a file for a service, I include them in the role/files folder, for example:

```yaml

/ansible/roles/blog
├── files
│   ├── favicon.png
│   ├── hugo.yaml
│   ├── og-image.png
│   └── terminal.css
├── tasks
│   └── main.yml
└── templates
    ├── blog-stack.yml.j2
    └── configs
        └── hugo.toml.j2

```

Above, I have themes files for my Hugo blog which are copied during the play to the hugo/static folder. Generally, I prefer to template files during the play, but in cases, such as the above, I don't need to change or edit the files and simply need them copied. 

***

## Role/Tasks

***

The role tasks contain everything required to automate what I need, with idempotence (that is, to be able to run the role as many times as I want and have it fulfill the tasks I set it out to do without fail). What I have found works best is to have as many tasks follow a streamlined and common structure as possible whenever you make a role. When deploying docker services, I have the following set of tasks:

***

### Clean Up

***

When carrying out tasks, it is best to bring down existing services. This is prevent any issues when making changes to running services. Also some changes will not be registered until a container or service is redeployed. 

**Bringing down a container:**

```yaml

- name: remove existing container
  community.docker.docker_container:
    container_default_behavior: compatibility
    name: '{{ radarr_name }}'
    state: absent
    stop_timeout: 10
  register: remove_radarr_docker
  retries: 5
  delay: 10
  until: remove_radarr_docker is succeeded

```

**Bringing down a Compose stack:**

```yaml

- name: Check if gluetun compose exists
  ansible.builtin.stat:
    path: '/opt/gluetun-compose.yml'
  register: gluetun_compose_yaml

- name: Remove gluetun compose stack
  when: gluetun_compose_yaml.stat.exists
  community.docker.docker_compose_v2:
    project_src: '/opt'
    files: gluetun-compose.yml
    state: absent

- name: Remove gluetun-compose file
  when: gluetun_compose_yaml.stat.exists
  ansible.builtin.file:
    path: '/opt/gluetun-compose.yml'
    state: absent

```

**Bringing down a Swarm stack:**

```yaml

- name: Remove arrs stack
  community.docker.docker_stack:
    name: arrs
    state: absent

- name: Remove arrs-stack file
  ansible.builtin.file:
    path: /opt/arrs-stack.yml
    state: absent

```

***

### Directories

***

I then create directories and sub-directories required for the role:

```yaml

- name: Create directories
  ansible.builtin.file:
    path: '{{ item }}'
    state: 'directory'
    force: 'false'
    owner: '{{ puid }}'
    group: '{{ pgid }}'
    mode: '0755'
  loop:
    - '{{ bazarr_location }}'
    - '{{ bazarr_location }}/config'
    - '{{ lidarr_location }}'
    - '{{ prowlarr_location }}'
    - '{{ radarr_location }}'
    - '{{ radarr_4k_location }}'
    - '{{ readarr_location }}'
    - '{{ sonarr_location }}'
    - '{{ sonarr_4k_location }}'
    - '{{ whisparr_location }}'
    - '{{ whisparr_v3_location }}'

```

Typically, the only thing differing between roles are directory paths, so it's a case of looping over directory paths. However, sometimes I require different permissions (i.e, bitnami), as below:

```yaml

- name: Create directories
  ansible.builtin.file:
    path: '{{ item.path }}'
    state: 'directory'
    force: 'false'
    owner: '{{ item.owner }}'
    group: '{{ item.group }}'
    mode: '0755'
  loop:
    - { path: '{{ authelia_location }}', owner: '{{ puid }}', group: '{{ pgid }}' }
    - { path: '{{ authelia_logs_location }}', owner: '{{ puid }}', group: '{{ pgid }}' }
    - { path: '{{ authelia_redis_location }}', owner: '1001', group: '1001' }
    - { path: '{{ traefik_location }}', owner: '{{ puid }}', group: '{{ pgid }}' }
    - { path: '{{ traefik_logs_location }}', owner: '{{ puid }}', group: '{{ pgid }}' }

```

Above, I loop over `path`, `owner`, `group`. However, you can name them whatever you want, as long as it matches up with the value after `item.`.

***

### Files

***

I deal with files using common tasks for [templates](https://drjoyce.blog/common/#templates) and [copy tasks](https://drjoyce.blog/common/#file-copy).

***

### Databases

***

Here I create postgres/mariadb databases using ansible modules:


**MariaDB**

I have a single task that checks for a database and creates one if none exists.

```yaml

- name: Create mariadb database
  community.mysql.mysql_db:
    login_host: '{{ pvr_machine }}'
    login_user: 'root'
    login_password: '{{ mariadb_password }}'
    login_port: '{{ mariadb_ports_host }}'
    name: 'wordpress'
    state: present

```

**Postgres**

I have [common tasks defined elsewhere](https://drjoyce.blog/common/).

Both require an existing mariadb/postgres database instance that is up and accepting connections, with all relevant database login details stored in `group_vars/vault`. I deploy my databases via docker, but it doesn't matter if you installed via a different method. Also, since these are ansible module tasks, and are not part of your docker network, they rely on machine ip and host port.

Once the database is created, depending on the services, additional database tasks will be required. These usually revolve around adding your database details to the service config. For me, I prefer defining the database details via the services docker environmental variables, but sometimes this isn't possible. Sometimes I prefer editing the config, which is the case with my arrs stack.

***

### Docker Secrets / Networks / Volumes

***

Here I create any required docker secrets, networks and volumes:

```yaml

- name: Prepare docker secrets
  community.docker.docker_secret:
    name: '{{ item.name }}'
    data: '{{ item.secret }}'
    state: present
  loop:
    - { name: 'unifi_user_secret', secret: '{{ unifi_db_user }}' }
    - { name: 'unifi_pass_secret', secret: '{{ unifi_db_pass }}' }
    - { name: 'unifi_root_pass_secret', secret: '{{ unifi_db_root_pass }}' }

```

```yaml

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

```yaml

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

***

### DNS

***

In this section, I add cloudflare DNS records via [common tasks](https://drjoyce.blog/common/#cloudflare-dns)


Previously, I would define the cloudflare tasks in role sub_tasks, like so:

```yaml

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

### Traefik

***

Here I include [common tasks](https://drjoyce.blog/common/#traefik-labels) to set Traefik labels for services.

***

### Deploy

***

The final step of tasks is to deploy the required container or stack:

**Deploying a container:**

```yaml

- name: create radarr container
  community.docker.docker_container:
    name: '{{ radarr_name }}'
    image: '{{ radarr_image_repo }}:{{ radarr_image_tag }}'
    networks: '{{ network_overlay }}'
    env:
      PUID: '{{ puid }}'
      PGID: '{{ pgid }}'
      TZ: '{{ timezone }}'
      TP_SCHEME: 'http'
      TP_DOMAIN: '{{ themepark_domain }}'
      TP_HOTIO: 'true'
      TP_THEME: '{{ themepark_theme }}'
    labels: '{{ radarr_labels }}'
    ports:
      - '{{ radarr_ports_host }}:{{ radarr_ports_cont }}'
    volumes:
      - '{{ radarr_location }}:/config'
      - '{{ themes_location }}/98-themepark-radarr:/etc/cont-init.d/98-themepark'
      - '/mnt:/mnt'
      - '{{ torrents_volume }}:{{ torrents_volume_mount_path }}'
    restart_policy: '{{ radarr_restart_policy }}'

```

**Deploying a Compose stack:**

```yaml

- name: Import gluetun compose file
  ansible.builtin.template:
    src: '{{ role_path }}/templates/gluetun-compose.yml.j2'
    dest: '/opt/gluetun-compose.yml'
    force: true
    owner: '{{ puid }}'
    group: '{{ pgid }}'
    mode: '0664'

- name: Gluetun compose stack up
  community.docker.docker_compose_v2:
    project_src: '/opt'
    files: gluetun-compose.yml
    state: present

```

**Deploying a Swarm stack:**

```yaml

- name: Import arrs-stack file
  ansible.builtin.template:
    src: '{{ role_path }}/templates/arrs-stack.yml.j2'
    dest: /opt/arrs-stack.yml
    force: true
    owner: '{{ puid }}'
    group: '{{ pgid }}'
    mode: '0664'

- name: Deploy arrs stack
  community.docker.docker_stack:
    state: present
    name: arrs
    compose:
      - /opt/arrs-stack.yml

```

***

## Sub-tasks

***











**Example:**

```

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



Sub_tasks prepare a docker swarm service to run, including:
  * Creating directories
  * Creating and preparing config files
  * Creating databases and adding database info to the config files 
  * Retrieving vault variables or generating keys/api/passwords during the play
  * Preparing docker secrets
  * Creating cloudflare A/CNAME DNS records
  and so on...



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