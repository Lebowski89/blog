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

Typically, the only thing differing between roles are directory paths, so it's a case of looping over directory paths. However, sometimes I require different permissions (i.e, bitnami), in which I iterate over a list of required hashes, as below:

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

After creating the directory, I typically deal with files via template and copy tasks.

**Copy**

For copies, [I have common tasks](https://drjoyce.blog/common/#file-copy) that get called

```yaml

- name: Copy Hugo Terminal Themes files
  ansible.builtin.include_tasks: /ansible/common/copy.yml
  vars:
    copy_source: '{{ item.source }}'
    copy_destination: '{{ item.destination }}'
  loop:
    - { source: '{{ role_path }}/files/favicon.png', destination: '/{{ hugo_site_name }}/static/favicon.png' }
    - { source: '{{ role_path }}/files/og-image.png', destination: '/{{ hugo_site_name }}/static/og-image.png' }
    - { source: '{{ role_path }}/files/terminal.css', destination: '/{{ hugo_site_name }}/static/terminal.css' }

```

**Templates**

For templates, like copies, [I have common tasks](https://drjoyce.blog/common/#templates) that get called:

```yaml

- name: Conduct template tasks
  ansible.builtin.include_tasks: '/ansible/common/template.yml'
  vars:
    template_location: '{{ item.template }}'
    file_location: '{{ item.file }}'
  loop:
    - { template: '{{ role_path }}/templates/configs/prometheus.yml.j2', file: '{{ prometheus_location }}/prometheus.yml' }
    - { template: '{{ role_path }}/templates/configs/loki-config.yaml.j2', file: '{{ loki_location }}/loki-config.yaml' }
    - { template: '{{ role_path }}/templates/configs/promtail-config.yaml.j2', file: '{{ promtail_location }}/promtail.yaml' }
    - { template: '{{ role_path }}/templates/configs/scraparr-config.yaml.j2', file: '{{ scraparr_location }}/config.yaml' }

```

Templates make up the majority of my files tasks, and it's how I handle config files for services.

***

### Databases

***

In this section I create any required postgres/mariadb databases using relevant ansible modules:


**MariaDB**

For MariaDB, I have a single task that checks for a database and creates one if none exists.

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

For Postgres I have [common tasks defined elsewhere](https://drjoyce.blog/common/) that is included using `ansible.builtin.include_tasks`. All that is required are the name of the databases.

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

Both cases require an existing mariadb/postgres database instance that is up and accepting connections, with all relevant database login details stored in `group_vars/vault`. I deploy my databases via docker, but it doesn't matter if you installed via a different method. Also, since these are ansible module tasks, and are not part of your docker network, they rely on your machine ip (gathered in pre-tasks) and host port.

Once the database is created, depending on the services, additional database tasks will be required. These usually revolve around adding your database details to the service config. For me, I prefer defining the database details via the services docker environmental variables, but sometimes this isn't possible. Sometimes I prefer editing the config, which is the case with my arrs stack:

1. I use the `ansible.builtin.include_tasks` to include some common postgres config tasks contained in my `arrs/tasks/sub_tasks` folder:

```yaml

- name: Conduct Postgres config tasks
  ansible.builtin.include_tasks: sub_tasks/postgres.yml
  vars:
    arrs_config_location: '{{ item.location }}'
    arrs_log_db: '{{ item.logdb }}'
    arrs_main_db: '{{ item.maindb }}'
    arrs_cache_db: '{{ item.cachedb }}'
  loop:
    - { location: '{{ lidarr_location }}', logdb: 'lidarr-log', maindb: 'lidarr-main', cachedb: '' }
    - { location: '{{ prowlarr_location }}', logdb: 'prowlarr-log', maindb: 'prowlarr-main', cachedb: '' }
    - { location: '{{ radarr_location }}', logdb: 'radarr-log', maindb: 'radarr-main', cachedb: '' }
    - { location: '{{ radarr_4k_location }}', logdb: 'radarr-4k-log', maindb: 'radarr-4k-main', cachedb: '' }
    - { location: '{{ readarr_location }}', logdb: 'readarr-log', maindb: 'readarr-main', cachedb: 'readarr-cache' }
    - { location: '{{ sonarr_location }}', logdb: 'sonarr-log', maindb: 'sonarr-main', cachedb: '' }
    - { location: '{{ sonarr_4k_location }}', logdb: 'sonarr-4k-log', maindb: 'sonarr-4k-main', cachedb: '' }
    - { location: '{{ whisparr_location }}', logdb: 'whisparr-log', maindb: 'whisparr-main', cachedb: '' }
    - { location: '{{ whisparr_v3_location }}', logdb: 'whisparr-log-v3', maindb: 'whisparr-main-v3', cachedb: '' }

```

2. The common tasks are conducted using the variables defined above:

```yaml

- name: Lookup PostgresUser value
  community.general.xml:
    path: '{{ arrs_config_location }}/config.xml'
    xpath: /Config/PostgresUser
    print_match: true
  register: arrs_xml_postgresuser

- name: Lookup PostgresPassword value
  community.general.xml:
    path: '{{ arrs_config_location }}/config.xml'
    xpath: /Config/PostgresPassword
    print_match: true
  register: arrs_xml_postgrespassword

- name: Lookup PostgresPort value
  community.general.xml:
    path: '{{ arrs_config_location }}/config.xml'
    xpath: /Config/PostgresPort
    print_match: true
  register: arrs_xml_postgresport

- name: Lookup PostgresHost value
  community.general.xml:
    path: '{{ arrs_config_location }}/config.xml'
    xpath: /Config/PostgresHost
    print_match: true
  register: arrs_xml_postgreshost

- name: Lookup PostgresMainDb value
  community.general.xml:
    path: '{{ arrs_config_location }}/config.xml'
    xpath: /Config/PostgresMainDb
    print_match: true
  register: arrs_xml_postgresmaindb

- name: Lookup PostgresLogDb value
  community.general.xml:
    path: '{{ arrs_config_location }}/config.xml'
    xpath: /Config/PostgresLogDb
    print_match: true
  register: arrs_xml_postgreslogdb

- name: Lookup PostgresCacheDb value
  community.general.xml:
    path: '{{ arrs_config_location }}/config.xml'
    xpath: /Config/PostgresCacheDb
    print_match: true
  register: arrs_xml_postgrescachedb

- name: Add PostgresUser
  when: ((arrs_xml_postgresuser.matches[0].PostgresUser is not defined) or
         (arrs_xml_postgresuser.matches[0].PostgresUser != postgres_username))
  community.general.xml:
    path: '{{ arrs_config_location }}/config.xml'
    pretty_print: true
    xpath: /Config/PostgresUser
    value: '{{ postgres_username }}'

- name: Add PostgresPassword
  when: ((arrs_xml_postgrespassword.matches[0].PostgresPassword is not defined) or
         (arrs_xml_postgrespassword.matches[0].PostgresPassword != postgres_password))
  community.general.xml:
    path: '{{ arrs_config_location }}/config.xml'
    pretty_print: true
    xpath: /Config/PostgresPassword
    value: '{{ postgres_password }}'

- name: Add PostgresPort
  when: ((arrs_xml_postgresport.matches[0].PostgresPort is not defined) or
         (arrs_xml_postgresport.matches[0].PostgresPort != postgres_ports_cont))
  community.general.xml:
    path: '{{ arrs_config_location }}/config.xml'
    pretty_print: true
    xpath: /Config/PostgresPort
    value: '{{ postgres_ports_cont }}'

- name: Add PostgresHost
  when: ((arrs_xml_postgreshost.matches[0].PostgresHost is not defined) or
         (arrs_xml_postgreshost.matches[0].PostgresHost != postgres_name))
  community.general.xml:
    path: '{{ arrs_config_location }}/config.xml'
    pretty_print: true
    xpath: /Config/PostgresHost
    value: '{{ postgres_name }}'

- name: Add PostgresMainDb
  when: ((arrs_xml_postgresmaindb.matches[0].PostgresMainDb is not defined) or
         (arrs_xml_postgresmaindb.matches[0].PostgresMainDb != arrs_main_db))
  community.general.xml:
    path: '{{ arrs_config_location }}/config.xml'
    pretty_print: true
    xpath: /Config/PostgresMainDb
    value: '{{ arrs_main_db }}'

- name: Add PostgresLogDb
  when: ((arrs_xml_postgreslogdb.matches[0].PostgresLogDb is not defined) or
         (arrs_xml_postgreslogdb.matches[0].PostgresLogDb != arrs_log_db))
  community.general.xml:
    path: '{{ arrs_config_location }}/config.xml'
    pretty_print: true
    xpath: /Config/PostgresLogDb
    value: '{{ arrs_log_db }}'

- name: Add PostgresCacheDb
  when: (arrs_cache_db is not none) and (arrs_cache_db | trim | length > 0)
  block:
    - name: Add PostgresCacheDb
      when: ((arrs_xml_postgrescachedb.matches[0].PostgresCacheDb is not defined) or
             (arrs_xml_postgrescachedb.matches[0].PostgresCacheDb != arrs_cache_db))
      community.general.xml:
        path: '{{ arrs_config_location }}/config.xml'
        pretty_print: true
        xpath: /Config/PostgresCacheDb
        value: '{{ arrs_cache_db }}'

```

3. I then have tasks to delete any existing sqlite databases in the arrs folder:

```yaml

- name: Remove sqlite files
  ansible.builtin.include_tasks: sub_tasks/sqlite.yml
  vars:
    sqlite_location: '{{ item.location }}'
    sqlite_name: '{{ item.name }}'
    backups_location: '{{ item.backups }}'
  loop:
    - { location: '{{ bazarr_location }}/db', name: 'bazarr', backups: '{{ bazarr_location }}/backup' }
    - { location: '{{ lidarr_location }}', name: 'lidarr', backups: '{{ lidarr_location }}/Backups' }
    - { location: '{{ prowlarr_location }}', name: 'prowlarr', backups: '{{ prowlarr_location }}/Backups' }
    - { location: '{{ radarr_location }}', name: 'radarr', backups: '{{ radarr_location }}/Backups' }
    - { location: '{{ radarr_4k_location }}', name: 'radarr', backups: '{{ radarr_4k_location }}/Backups' }
    - { location: '{{ readarr_location }}', name: 'readarr', backups: '{{ readarr_location }}/Backups' }
    - { location: '{{ sonarr_location }}', name: 'sonarr', backups: '{{ sonarr_location }}/Backups' }
    - { location: '{{ sonarr_4k_location }}', name: 'sonarr', backups: '{{ sonarr_4k_location }}/Backups' }
    - { location: '{{ whisparr_location }}', name: 'whisparr', backups: '{{ whisparr_location }}/Backups' }
    - { location: '{{ whisparr_v3_location }}', name: 'whisparr', backups: '{{ whisparr_v3_location }}/Backups' }

```

```yaml

- name: Remove sqlite files
  ansible.builtin.file:
    path: '{{ item }}'
    state: absent
  loop:
    - '{{ sqlite_location }}/logs.db'
    - '{{ sqlite_location }}/logs.db-shm'
    - '{{ sqlite_location }}/logs.db-wal'
    - '{{ sqlite_location }}/{{ sqlite_name }}.db'
    - '{{ sqlite_location }}/{{ sqlite_name }}.db-shm'
    - '{{ sqlite_location }}/{{ sqlite_name }}.db-wal'

- name: Remove sqlite backups folders
  ansible.builtin.file:
    path: '{{ backups_location }}'
    state: absent

```

Of course, if you still need your sqlite databases you wouldn't use these. But for me, I've already fully migrated to postgres, so the only time an sqlite file would show up is if something went wrong with the role. 

***

### DNS

***

In this section, I add cloudflare DNS records for any services that require it via [common tasks](https://drjoyce.blog/common/#cloudflare-dns)

```yaml

- name: Add DNS records
  ansible.builtin.include_tasks: /ansible/common/cloudflare.yml
  vars:
    cloudflare_domain: '{{ local_domain }}'
    cloudflare_record: '{{ item }}'
    cloudflare_type: 'A'
    cloudflare_value: '{{ ipify_public_ip }}'
    cloudflare_proxy: 'false'
    cloudflare_solo: 'true'
    cloudflare_remove_existing: 'true'
  loop:
    - '{{ prometheus_name }}'
    - '{{ sabnzbd_name }}'

```

Prior to moving to common tasks, I would define the cloudflare tasks in role sub_tasks, like so:

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

In this section, include [common tasks](https://drjoyce.blog/common/#traefik-labels) to set Traefik labels for each services requiring them:

```yaml

- name: Set traefik Labels
  ansible.builtin.include_tasks: /ansible/common/labels.yml
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
    ## grafana
    - { var: 'grafana_labels',
        name: '{{ grafana_name }}',
        port: '{{ grafana_ports_cont }}',
        api: '',
        sso: ',authelia@swarm',
        tp: '',
        tp_app: '' }
    ## prometheus
    - { var: 'prometheus_labels',
        name: '{{ prometheus_name }}',
        port: '{{ prometheus_ports_cont }}',
        api: '',
        sso: ',authelia@swarm',
        tp: '',
        tp_app: '' }

```


   - Deploy














Removes existing stacks, retrieves common vault variables, includes sub_tasks, and then deploys the stack.

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