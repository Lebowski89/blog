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


Previously, I would define the cloudflare tasks in the role, like so:

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

Here I use [common tasks](https://drjoyce.blog/common/#traefik-labels) to set Traefik labels for services.

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

## Roles/Sub-tasks

***

I make use of sub-tasks in roles if a set of tasks meet the following criteria:
   
   1. The tasks are role specific/niche and not appropriate for common tasks
   2. The number of tasks would clutter up the main tasks file.

One example is my recyclarr config tasks:

```yaml

- name: Include recyclarr tasks
  ansible.builtin.include_tasks: sub_tasks/recyclarr.yml

```

```yaml

################################
# CONFIG
################################

- name: Check config exists
  ansible.builtin.stat:
    path: '{{ recyclarr_location }}/recyclarr.yml'
  register: recyclarr_config

- name: Generate default config
  when: not recyclarr_config.stat.exists
  community.docker.docker_container:
    name: recyclarr-config
    image: '{{ recyclarr_image_repo }}:{{ recyclarr_image_tag }}'
    pull: true
    state: started
    container_default_behavior: compatibility
    networks:
      - name: '{{ network_overlay }}'
    env:
      RECYCLARR_CREATE_CONFIG: 'true'
    user: '{{ puid }}:{{ pgid }}'
    volumes:
      - '{{ recyclarr_location }}:/config'
    cleanup: true

- name: Remove config container
  community.docker.docker_container:
    container_default_behavior: compatibility
    name: recyclarr-config
    state: absent
    stop_timeout: 10
  register: remove_recyclarr_config_container
  retries: 5
  delay: 10
  until: remove_recyclarr_config_container is succeeded

################################
# SECRETS
################################

- name: Insert recyclarr secrets settings
  ansible.builtin.blockinfile:
    path: '{{ recyclarr_location }}/secrets.yml'
    marker: "## {mark} ANSIBLE RECYCLARR MANAGED BLOCK ##"
    owner: '{{ puid }}'
    group: '{{ pgid }}'
    mode: '0664'
    create: true
    block: |
      radarr_base_url: http://{{ radarr_name }}:{{ radarr_ports_cont }}
      radarr_api_key: {{ radarr_api }}
      radarr4k_base_url: http://{{ radarr_4k_name }}:{{ radarr_4k_ports_cont }}
      radarr4k_api_key: {{ radarr_4k_api }}
      sonarr_base_url: http://{{ sonarr_name }}:{{ sonarr_ports_cont }}
      sonarr_api_key: {{ sonarr_api }}
      sonarr4k_base_url: http://{{ sonarr_4k_name }}:{{ sonarr_4k_ports_cont }}
      sonarr4k_api_key: {{ sonarr_4k_api }}

################################
# CONFIGS
################################

  ## This is just one of the configs. I have another three.

- name: Insert radarr config settings
  ansible.builtin.blockinfile:
    path: '{{ recyclarr_location }}/configs/radarr.yml'
    marker: "## {mark} ANSIBLE RADARR MANAGED BLOCK ##"
    owner: '{{ puid }}'
    group: '{{ pgid }}'
    mode: '0664'
    create: true
    block: |
      radarr:
        radarr:
          media_naming:
            folder: plex
            movie:
              rename: true
              standard: default
          quality_definition:
            type: movie
          quality_profiles:
            - name: Remux + WEB 1080p
              reset_unmatched_scores:
                enabled: true
              upgrade:
                allowed: true
                until_quality: Remux-1080p
                until_score: 10000
              min_format_score: 0
              quality_sort: top
              qualities:
                - name: Remux-1080p
                - name: Bluray-1080p
                  enabled: false
                - name: WEB 1080p
                  qualities:
                    - WEBRip-1080p
                    - WEBDL-1080p
          delete_old_custom_formats: true
          replace_existing_custom_formats: true
          custom_formats:
            - trash_ids:
                  # Audio Advanced #1
                - 417804f7f2c4308c1f4c5d380d4c4475 # ATMOS (undefined)
                - 1af239278386be2919e1bcee0bde047e # DD+ ATMOS
                - 185f1dd7264c4562b9022d963ac37424 # DD+
                - f9f847ac70a0af62ea4a08280b859636 # DTS-ES
                - dcf3ec6938fa32445f590a4da84256cd # DTS-HD MA
                - 2f22d89048b01681dde8afe203bf2e95 # DTS X
                - 1c1a4c5e823891c75bc50380a6866f73 # DTS
                - 496f355514737f7d83bf7aa4d24f8169 # TrueHD ATMOS
                - 3cafb66171b47f226146a0770576870f # TrueHD
                  # Audio Advanced #2
                - 240770601cc226190c367ef59aba7463 # AAC
                - c2998bd0d90ed5621d8df281e839436e # DD
                - 8e109e50e0a0b83a5098b056e13bf6db # DTS-HD HRA
                - a570d4a0e56a2874b64e5bfa55202a1b # FLAC
                - 6ba9033150e7896bdc9ec4b44f2b230f # MP3
                - a061e2e700f81932daf888599f8a8273 # Opus
                - e7c2fcae07cbada050a0af3357491d7b # PCM
                  # HDR Formats
                - e23edd2482476e595fb990b12e7c609c # DV HDR10
                - 55d53828b9d81cbe20b02efd00aa0efd # DV HLG
                - a3e19f8f627608af0211acd02bf89735 # DV SDR
                - 58d6a88f13e2db7f5059c41047876f00 # DV
                - 2a4d9069cc1fe3242ff9bdaebed239bb # HDR (undefined)
                - e61e28db95d22bedcadf030b8f156d96 # HDR
                - dfb86d5941bc9075d6af23b09c2aeecd # HDR10
                - b974a6cd08c1066250f1f177d7aa1225 # HDR10+
                - 9364dd386c9b4a1100dde8264690add7 # HLG
                - 08d6d8834ad9ec87b1dc7ec8148e7a1f # PQ
                  # HQ Release Groups
                - 3a3ff47579026e76d6504ebea39390de # Remux Tier 01
                - 9f98181fe5a3fbeb0cc29340da2a468a # Remux Tier 02
                - 8baaf0b3142bf4d94c42a724f034e27a # Remux Tier 03
                - 403816d65392c79236dcb6dd591aeda4 # WEB Tier 02
                - c20f169ef63c5f40c2def54abaf4438e # WEB Tier 01
                - af94e0fe497124d1f9ce732069ec8c3b # WEB Tier 03
                  # Movie Versions
                - eca37840c13c6ef2dd0262b141a5482f # 4K Remaster
                - e0c07d59beb37348e975a930d5e50319 # Criterion Collection
                - 0f12c086e289cf966fa5948eac571f44 # Hybrid
                - 9f6cbff8cfe4ebbc1bde14c7b7bec0de # IMAX Enhanced
                - eecf3a857724171f968a66cb5719e152 # IMAX
                - 570bc9ebecd92723d2d21500f4be314c # Remaster
                - 957d0f44b592285f26449575e8b1167e # Special Edition
                - e9001909a4c88013a359d0b9920d7bea # Theatrical Cut
                - 9d27d9d2181838f76dee150882bdc58c # Masters of Cinema
                - 09d9dd29a0fc958f9796e65c2a8864b4 # Open Matte
                - db9b4c4b53d312a3ca5f1378f6440fc9 # Vinegar Syndrome
                  # Unwanted
                - cae4ca30163749b891686f95532519bd # AV1
                - ed38b889b31be83fda192888e2286d83 # BR-DISK
                - e6886871085226c3da1830830146846c # Generated Dynamic HDR
                - 90a6f9a284dff5103f6346090e6280c8 # LQ
                - e204b80c87be9497a8a6eaff48f72905 # LQ (Release Title)
                - 712d74cd88bceb883ee32f773656b1f5 # Sing-Along Versions
                - b8cd450cbfa689c0259a01d9e29ba3d6 # 3D
                - dc98083864ea246d05a42df0d05f81cc # x265 (HD)
                - bfd8eb01832d646a0a89c4deb46f8564 # Upscaled
                - 0a3f082873eb454bde444150b70253cc # Extras
                  # Misc
                - b6832f586342ef70d9c128d40c07b872 # Bad Dual Groups
                - ae9b7c9ebde1f3bd336a8cbd1ec4c5e5 # No-RlsGroup
                - 5c44f52a8714fdd79bb4d98e2673be1f # Retags
                - 182fa1c42a2468f8488e6dcf75a81b81 # Internal
                - c465ccc73923871b3eb1802042331306 # Line/Mic Dubbed
                - e7718d7a3ce595f289bfee26adc178f5 # Repack/Proper
                - ae43b294509409a6a13919dedd4764c4 # Repack2
                - 5caaaa1c08c1742aa4342d8c4cc463f2 # Repack3
                - 2899d84dc9372de3408e6d8cc18e9666 # x264
                - 0d91270a7255a1e388fa85e959f359d8 # FreeLeech
                  # Streaming Services
                - b3b3a6ac74ecbd56bcdbefa4799fb9df # AMZN
                - 40e9380490e748672c2522eaaeb692f7 # ATVP
                - 84272245b2988854bfb76a16e60baea5 # DSNP
                - 509e5f41146e278f9eab1ddaceb34515 # HBO
                - 5763d1b0ce84aff3b21038eea8e9b8ad # HMAX
                - 526d445d4c16214309f0fd2b3be18a89 # Hulu
                - e0ec9672be6cac914ffad34a6b077209 # iT
                - 6a061313d22e51e0f25b7cd4dc065233 # MAX
                - 170b1d363bd8516fbf3a3eb05d4faff6 # NF
                - c9fd353f8f5f1baf56dc601c4cb29920 # PCOK
                - e36a0ba1bc902b26ee40818a1d59b8bd # PMTP
                - c2863d2a50c9acad1fb50e53ece60817 # STAN
              assign_scores_to:
                - name: Remux + WEB 1080p

```

The idea is to keep the main tasks file as streamlined as possible.

```

***

## Roles/Templates

***

As the name suggests, the roles templates folder is where I keep all the files that will be templated during the play.

***

### Configs

***

Most roles will include at least one config file to be templated during the play. 

I find templating more powerful than simply copying, as a change in a variable's value will apply to the config template referencing it, negating the need to edit config files individually. 

**Example:**

```yaml

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

***

### Compose

***

Each role will also have a docker compose file, containing the roles docker services.

**Docker Compose**

```yaml

version: '3.9'

services:
  {{ gluetun_name }}:
    container_name: {{ gluetun_name }}
    image: {{ gluetun_image_repo }}:{{ gluetun_image_tag }}
    networks:
      - {{ network_overlay }}
    devices:
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
    environment:
      PUID: '{{ puid }}'
      PGID: '{{ pgid }}'
      TZ: '{{ timezone }}'
      ## And various other gluetun env...
    ports:
      - {{ jd2_ports_host }}:{{ jd2_ports_cont }}/tcp
      - {{ searxng_ports_host }}:{{ searxng_ports_cont }}/tcp
      - {{ librewolf_ports_http_host }}:{{ librewolf_ports_http_cont }}/tcp
      - {{ librewolf_ports_https_host }}:{{ librewolf_ports_https_cont }}/tcp
    volumes:
      - {{ gluetun_location }}:/gluetun

  {{ jd2_name }}:
    container_name: {{ jd2_name }}
    depends_on:
      - {{ gluetun_name }}
    image: {{ jd2_image_repo }}:{{ jd2_image_tag }}
    network_mode: 'service:{{ gluetun_name }}'
    environment:
      USER_ID: '{{ puid }}'
      GROUP_ID: '{{ pgid }}'
      TZ: '{{ timezone }}'
    volumes:
      - {{ jd2_location }}:/config
      - type: bind
        source: /mnt
        target: /output

```

**Docker Swarm**

```yaml

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

```

In my case, I prefer letting Ansible docker modules handle networks / secrets / volumes, so they're listed as external at the bottom of the compose file, like so:

```yaml

networks:
  {{ network_overlay }}:
    external: true

secrets:
  unifi_user_secret:
    external: true

  unifi_pass_secret:
    external: true

  unifi_root_pass_secret:
    external: true

volumes:
  {{ media_volume }}:
    external: true

```