---
draft: false
genres:
- ansible
menus: common
weight: 1
title: Handling common tasks within Ansible
---

***

## Introduction

***

When you begin constructing your Ansible tasks - roles - collections, you quickly realise that there are many tasks that you will use repeatedly with little variance. For example, if you're using Ansible to help set-up and deploy your docker containers, you'll have common tasks that create directories, configs, DNS and Traefik labels, among many others.

For me, my common tasks consist of two main types:

1) Multiple (3+) common tasks defined in a file in a common tasks folder (external to the role), which are included in the role using the `include_tasks` module
2) Smaller (1-3) common tasks defined and confined entirely within roles.

The rest of this document is dedicated to listing and describing common tasks that I use:

***

## Cloudflare DNS

***

One time-saving task when spinning up docker services is to automate the creationa and removal of cloudflare DNS records.

```yaml

  ## ansible/common/cloudflare.yml

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

```yaml

  ## role/tasks/main.yml

- name: Add Cloudflare DNS records
  ansible.builtin.include_tasks: '/ansible/common/cloudflare.yml'
  vars:
    cloudflare_domain: '{{ local_domain }}'
    cloudflare_record: '{{ obsidian_name }}'
    cloudflare_type: 'A'
    cloudflare_value: '{{ ipify_public_ip }}'
    cloudflare_proxy: false
    cloudflare_solo: true
    cloudflare_remove_existing: true
```

Above, variables from the common task are replaced with relevant values required to create a DNS record for my Obsidian container running on my local server. However, you can create loops to handle as many services as you require:

```yaml

  ## role/tasks/main.yml

- name: Add 'GitHub Pages' DNS records
  ansible.builtin.include_tasks: '/ansible/common/cloudflare.yml'
  vars:
    cloudflare_domain: '{{ github_pages_domain }}'
    cloudflare_record: '@'
    cloudflare_type: '{{ item.type }}'
    cloudflare_value: '{{ item.value }}'
    cloudflare_proxy: false
    cloudflare_solo: false
    cloudflare_remove_existing: false
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

***

## Directories and Files

***

I create directories required by roles with the `ansible.builtin.file` module:

```yaml

  ## role/tasks/main.yml

- name: Create directories
  ansible.builtin.file:
    path: '{{ item }}'
    state: directory
    force: false
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
    - '{{ sonarr_location }}'
    - '{{ sonarr_4k_location }}'
    - '{{ whisparr_location }}'

```

Sometimes I'll need different permissions for directories in the loop:

```yaml

  ## role/tasks/main.yml

- name: Create directories
  ansible.builtin.file:
    path: '{{ item.path }}'
    state: directory
    force: false
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

Above, I use the Bitnami Redis container, which requires 1001/1001 permissions.

The `builtin.file` module is also useful when you need to 'touch' a file:

```yaml

  ## role/tasks/main.yml

- name: Touch acme.json
  ansible.builtin.file:
    path: '{{ traefik_location }}/acme.json'
    state: touch
    force: false
    owner: '{{ puid }}'
    group: '{{ pgid }}'
    mode: '0600'

```

Removing directories and files is as simple as providing the `builtin.file` module a path and `state: absent`:

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

***

## File Copy

***

A simple task to copy files from one directory to another. I typically use this to copy files that docker services require from their role folder to the services appdata directory (though, I prefer to template files where and when possible).

```yaml

  ## ansible/common/copy.yml

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

```yaml

  ## role/tasks/main.yml

- name: Copy Hugo Terminal Themes files
  ansible.builtin.include_tasks: '/ansible/common/copy.yml'
  vars:
    copy_source: '{{ item.source }}'
    copy_destination: '{{ item.destination }}'
    copy_force: true
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

A core reason I'm using Ansible is to set up configs for my services. For this, I typically template a config file from the role into the service folder.


```yaml

  ## role/tasks/main.yml

- name: Conduct template tasks
  ansible.builtin.template:
    src: '{{ item.template }}'
    dest: '{{ item.file }}'
    force: false
    owner: '{{ puid }}'
    group: '{{ pgid }}'
    mode: '0664'
  loop:
    - { template: '{{ role_path }}/templates/configs/bazarr_config.yaml.j2', file: '{{ bazarr_location }}/config/config.yaml' }
    - { template: '{{ role_path }}/templates/configs/arrs_config.xml.j2', file: '{{ lidarr_location }}/config.xml' }
    - { template: '{{ role_path }}/templates/configs/arrs_config.xml.j2', file: '{{ prowlarr_location }}/config.xml' }
    - { template: '{{ role_path }}/templates/configs/arrs_config.xml.j2', file: '{{ radarr_location }}/config.xml' }
    - { template: '{{ role_path }}/templates/configs/arrs_config.xml.j2', file: '{{ radarr_4k_location }}/config.xml' }
    - { template: '{{ role_path }}/templates/configs/arrs_config.xml.j2', file: '{{ sonarr_location }}/config.xml' }
    - { template: '{{ role_path }}/templates/configs/arrs_config.xml.j2', file: '{{ sonarr_4k_location }}/config.xml' }
    - { template: '{{ role_path }}/templates/configs/arrs_config.xml.j2', file: '{{ whisparr_location }}/config.xml' }

- name: Wait for files to be created
  ansible.builtin.wait_for:
    path: '{{ item }}'
    state: present
  loop:
    - '{{ bazarr_location }}/config/config.yaml'
    - '{{ lidarr_location }}/config.xml'
    - '{{ prowlarr_location }}/config.xml'
    - '{{ radarr_location }}/config.xml'
    - '{{ radarr_4k_location }}/config.xml'
    - '{{ sonarr_location }}/config.xml'
    - '{{ sonarr_4k_location }}/config.xml'
    - '{{ whisparr_location }}/config.xml'
```

Above, I template configs from the arrs role folder into respective arrs appdata directories. The `wait_for` task ensures templated files are in place before subsequent tasks continue.

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

Traefik is my reverse-proxy of choice, with much of the config taking place in docker labels. Previously, I would define the labels for each service individually in role defaults. Depending on the service, I have http, https, api and theme-park labels. It can get extensive, but most of the labels are common across services with few differing variables.

To reduce the bloat, I make use of:
   1. A labels template contained in my group_vars
   2. A common task file to determine required labels
   3. Include_tasks task with relevant variables.


```yaml

  ## ansible/group_vars/all/traefik.yml

traefik_middlewares: 'globalHeaders@file,autodetect@swarm,gzip@swarm,robotHeaders@file'
traefik_http_middlewares: '{{ traefik_middlewares + ",redirect-to-https@swarm,cloudflarewarp@swarm" }}'
traefik_https_middlewares: '{{ traefik_middlewares + ",secureHeaders@file,hsts@file,cloudflarewarp@swarm" }}'

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

```yaml

  ## ansible/common/labels.yml

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

```yaml

  ## role/tasks/main.yml

- name: Set traefik Labels
  ansible.builtin.include_tasks: /ansible/common/labels.yml
  vars:
    router_variable: '{{ item.var }}'
    router_network: '{{ network_overlay }}'
    router_name: '{{ item.name }}'
    router_port: '{{ item.port }}'
    router_api: '{{ item.api }}'
    router_domain: '{{ local_domain }}'
    router_entrypoint: 'http'
    router_secure_entrypoint: 'https'
    router_tls_certresolver: 'dns-cloudflare'
    router_tls_options: 'securetls@file'
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

In the above example, both services receive full Traefik labels, including being protected by my SSO provider (Authelia), API labels and Theme-Park. Simply leaving the API and/or Themepark vars empty will remove these labels.

***

## Databases

***

I let Ansible handle the creation of various databases required by my services:


***

### Postgres

***

I use the postgresql ping and db modules to ping for databases and to create them if none exists:

**Note:** Requires an existing running postgres instance.

```yaml

  ## ansible/common/postgres.yml

- name: Ping for existing database
  community.postgresql.postgresql_ping:
    login_host: '{{ local_ip }}'
    login_user: '{{ postgres_username }}'
    login_password: '{{ postgres_password }}'
    port: '{{ postgres_ports_host }}'
    login_db: '{{ postgres_database }}'
  register: postgres_db_exists

- name: Create postgres database
  when: not postgres_db_exists == true
  community.postgresql.postgresql_db:
    login_host: '{{ local_ip }}'
    login_user: '{{ postgres_username }}'
    login_password: '{{ postgres_password }}'
    port: '{{ postgres_ports_host }}'
    name: '{{ postgres_database }}'
    state: present

```

```yaml

  ## role/tasks/main.yml

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
    - 'sonarr-main'
    - 'sonarr-log'
    - 'sonarr-4k-main'
    - 'sonarr-4k-log'
    - 'whisparr-main'
    - 'whisparr-log'

```

Above, the only the database name changes, since I'm working with one postgres instance.

***

### MariaDB

***

I use the `mysql_db` module to ping and create databases in a single task:

```yaml

  ## role/tasks/main.yml

- name: Create mariadb database
  community.mysql.mysql_db:
    login_host: '{{ local_machine }}'
    login_user: 'root'
    login_password: '{{ mariadb_password }}'
    login_port: '{{ mariadb_ports_host }}'
    name: wordpress
    state: present

```

**Note:** Requires an existing MariaDB instance to be present and running.

***

## Docker

***

I use Ansible to automate important Docker tasks:

***

### Docker Containers

***

Removing individual containers using the `docker_container` module:

```yaml

  ## role/tasks/main.yml

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

Creating containers using the same module:

```yaml

  ## role/tasks/main.yml

- name: Create radarr container
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

***

### Docker Compose

***

Compose down using the `docker_compose_v2` module:

```yaml

  ## role/tasks/main.yml

- name: Check if VPN compose exists
  ansible.builtin.stat:
    path: /opt/vpn-compose.yml
  register: vpn_compose_yaml

- name: VPN compose down
  when: vpn_compose_yaml.stat.exists
  community.docker.docker_compose_v2:
    project_src: /opt
    files: vpn-compose.yml
    state: absent

- name: Remove vpn-compose file
  when: vpn_compose_yaml.stat.exists
  ansible.builtin.file:
    path: /opt/vpn-compose.yml
    state: absent

```

Compose up using the same module:

```yaml

  ## role/tasks/main.yml

- name: Import VPN compose file
  ansible.builtin.template:
    src: '{{ role_path }}/templates/vpn-compose.yml.j2'
    dest: /opt/vpn-compose.yml
    force: true
    owner: '{{ puid }}'
    group: '{{ pgid }}'
    mode: '0664'

- name: VPN compose up
  community.docker.docker_compose_v2:
    project_src: /opt
    files: vpn-compose.yml
    state: present

```

***

### Docker Stacks

***

Docker stack down using the `docker_stack` module:

```yaml

  ## role/tasks/main.yml

- name: Remove arrs stack
  community.docker.docker_stack:
    name: arrs
    state: absent

- name: Remove arrs-stack file
  ansible.builtin.file:
    path: /opt/arrs-stack.yml
    state: absent

```

Stack deploy using the same module:

```yaml

  ## role/tasks/main.yml

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

### Docker Secrets

***

Creating Docker Secrets using the `docker_secret` module:

```yaml

  ## role/tasks/main.yml

- name: Prepare docker secrets
  community.docker.docker_secret:
    name: '{{ item.name }}'
    data: '{{ item.secret }}'
    state: present
    force: false
  loop:
    - { name: 'authelia_redis_secret', secret: '{{ authelia_redis_key }}' }
    - { name: 'postgres_pass_secret', secret: '{{ postgres_password }}' }

```

Then referring to these in the compose file:

```yaml

services:
  {{ authelia_name }}:
    environment:
      AUTHELIA_SESSION_REDIS_PASSWORD_FILE: /run/secrets/authelia_redis_secret
      AUTHELIA_STORAGE_POSTGRES_PASSWORD_FILE: /run/secrets/postgres_pass_secret
    secrets:
      - authelia_redis_secret
      - postgres_pass_secret

secrets:
  authelia_redis_secret:
    external: true

  postgres_pass_secret:
    external: true

```

<script data-name="BMC-Widget" data-cfasync="false" src="https://cdnjs.buymeacoffee.com/1.0.0/widget.prod.min.js" data-id="lebowski89" data-description="Support me on Buy me a coffee!" data-message="Support Me" data-color="#5F7FFF" data-position="Right" data-x_margin="18" data-y_margin="18"></script>