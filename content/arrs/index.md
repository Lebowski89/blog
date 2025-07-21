---
draft: false
genres:
- ansible
menus: arrs
weight: 1
title: Deploying an arrs stack Ansible Role
---

***

## Introduction

***

Some of my favourite services to deploy are the servarr / arrs PVR apps:

   - Bazarr (Subtitles)
{{< image src="bazarr_1.jpg" alt="bazarr" position="center"
    style="border-radius: 1px; width:auto; height: auto" >}}
   - Lidarr (Music)
{{< image src="lidarr_1.jpg" alt="lidarr" position="center"
    style="border-radius: 1px; width:auto; height: auto" >}}
   - Radarr (Movies) / Radarr-4K (4K Movies)
{{< image src="radarr_1.jpg" alt="radarr" position="center"
    style="border-radius: 1px; width:auto; height: auto" >}}
   - Sonarr (TV) / Sonarr-4K (4K TV)
{{< image src="sonarr_1.jpg" alt="sonarr" position="center"
    style="border-radius: 1px; width:auto; height: auto" >}}
   - Whisparr (Adult)
{{< image src="whisparr_1.jpg" alt="whisparr" position="center"
    style="border-radius: 1px; width:auto; height: auto" >}}
   - Prowlarr (Indexers)
{{< image src="prowlarr_1.jpg" alt="prowlarr" position="center"
    style="border-radius: 1px; width:auto; height: auto" >}}

Using Ansible, I do everything required to deploy these services for my needs.

This guide will: 
   - Go through my arrs role step-by-step, from clean-up to deployment.
   - Aim to guide those looking to make their own role for their own needs.

***

## Playbook

***

([Background Information](https://drjoyce.blog/playbook/))

I include my arrs services in a role that gets included during the play:

```yaml

  ## /ansible/playbook.yml

################################
# ARRS
################################

    - name: Deploy arrs stack
      when: inventory_hostname == 'localhost'
      ansible.builtin.include_role:
        name: arrs
        apply:
          tags: arrs
      tags: arrs

```

**Note:** I make use of the pre-tasks listed [here](https://drjoyce.blog/playbook/#pre_tasks). You may use these, or you can always define things, such as timezones, manually if you'd prefer.

***

## Group_Vars

***

([Background Information](https://drjoyce.blog/roles/#group_vars))

Here I define variables for service name, image repo/tag, ports and location:

```yaml

  ## /ansible/group_vars/all/roles/arrs.yml

################################
# BAZARR
################################

bazarr_name: 'bazarr'
bazarr_image_repo: 'ghcr.io/hotio/bazarr'
bazarr_image_tag: 'latest'
bazarr_ports_host: '6767'
bazarr_ports_cont: '6767'
bazarr_location: '/opt/{{ bazarr_name }}'

################################
# LIDARR
################################

lidarr_name: 'lidarr'
lidarr_image_repo: 'ghcr.io/hotio/lidarr'
lidarr_image_tag: 'latest'
lidarr_ports_host: '8686'
lidarr_ports_cont: '8686'
lidarr_location: '/opt/{{ lidarr_name }}'

################################
# PROWLARR
################################

prowlarr_name: 'prowlarr'
prowlarr_image_repo: 'ghcr.io/hotio/prowlarr'
prowlarr_image_tag: 'latest'
prowlarr_ports_host: '9696'
prowlarr_ports_cont: '9696'
prowlarr_location: '/opt/{{ prowlarr_name }}'

################################
# RADARR
################################

radarr_name: 'radarr'
radarr_image_repo: 'ghcr.io/hotio/radarr'
radarr_image_tag: 'latest'
radarr_ports_host: '7878'
radarr_ports_cont: '7878'
radarr_location: '/opt/{{ radarr_name }}'

################################
# RADARR-4K
################################

radarr_4k_name: 'radarr4k'
radarr_4k_image_repo: 'ghcr.io/hotio/radarr'
radarr_4k_image_tag: 'latest'
radarr_4k_ports_host: '7879'
radarr_4k_ports_cont: '7878'
radarr_4k_location: '/opt/{{ radarr_4k_name }}'

################################
# SONARR
################################

sonarr_name: 'sonarr'
sonarr_image_repo: 'ghcr.io/hotio/sonarr'
sonarr_image_tag: 'latest'
sonarr_ports_host: '8989'
sonarr_ports_cont: '8989'
sonarr_location: '/opt/{{ sonarr_name }}'

################################
# SONARR-4K
################################

sonarr_4k_name: 'sonarr4k'
sonarr_4k_image_repo: 'ghcr.io/hotio/sonarr'
sonarr_4k_image_tag: 'latest'
sonarr_4k_ports_host: '8990'
sonarr_4k_ports_cont: '8989'
sonarr_4k_location: '/opt/{{ sonarr_4k_name }}'

################################
# WHISPARR
################################

whisparr_name: 'whisparr'
whisparr_image_repo: 'ghcr.io/hotio/whisparr'
whisparr_image_tag: 'v3'
whisparr_ports_host: '6969'
whisparr_ports_cont: '6969'
whisparr_location: '/opt/{{ whisparr_name }}'

```

***

### Vault

***

```yaml

  ## /ansible/group_vars/all/vault.yml

bazarr_api: 'SomeAPIKey'
bazarr_flask_key: 'SomeAPIKey'
bazarr_opensub_user: 'SomeUsername'
bazarr_opensub_pass: 'SomePassword'
lidarr_api: 'SomeAPIKey'
prowlarr_api: 'SomeAPIKey'
radarr_api: 'SomeAPIKey'
radarr_4k_api: 'SomeAPIKey'
sonarr_api: 'SomeAPIKey'
sonarr_4k_api: 'SomeAPIKey'
whisparr_api: 'SomeAPIKey'

  ## If wanting Postgres databases:

postgres_username: 'SomeUsername'
postgres_password: 'SomePassword'

  ## If wanting DNS / Traefik:

local_domain: 'SomeDomain.com'

```

***

### Others

***

```yaml

  ## /ansible/group_vars/all/docker.yml

puid: '1000'
pgid: '1000'

network_overlay: 'overlay'  ## The variable I use for my docker network name

  ## If wanting to deploy theme-park locally:

themepark_name: 'theme-park'
themepark_ports_host: '8089'
themepark_location: '/opt/{{ themepark_name }}'
themepark_domain: '{{ local_ip + ":" + themepark_ports_host }}'
themepark_theme: 'hotline'  ## theme to use for local theme-park deployment
themes_location: '{{ themepark_location }}/docker-mods'

```

I define these in group_vars, but you could just as easily define them manually.

***

## Role Structure

***

My arrs role consists of 5 directories and 7 files:

```cli

/ansible/roles/arrs
├── tasks
│   ├── main.yml
│   └── sub_tasks
│       ├── config.yml
│       ├── postgres.yml
│       └── sqlite.yml
└── templates
    ├── arrs-stack.yml.j2
    └── configs
        ├── arrs_config.xml.j2
        └── bazarr_config.yaml.j2

```

***

## Role Tasks

***

([Background Information](https://drjoyce.blog/roles/#roletasks))


***

### Cleanup

***

I first down existing running arrs services:

([Click for overview](https://drjoyce.blog/common/#stacks))

```yaml

  ## /ansible/roles/arrs/tasks/main.yml

################################
# CLEAN UP
################################

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

Next is to create appdata folders for each arrs service.

Bazarr differs with the config located in a config sub-directory.

For those wishing to use Postgres, these folders are simply used to hold configs.

([Click for overview](https://drjoyce.blog/common/#directories-and-files))


```yaml

################################
# DIRECTORIES
################################

  ## /ansible/roles/arrs/tasks/main.yml

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
    - '{{ sonarr_location }}'
    - '{{ sonarr_4k_location }}'
    - '{{ whisparr_location }}'

```

***

### Templates

***

After the directories are created, I then template configs for each arrs service:

([Click for overview](https://drjoyce.blog/common/#templates))

```yaml

  ## /ansible/roles/arrs/tasks/main.yml

################################
# TEMPLATES
################################

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

All arrs, except Bazarr, use the same config template:

```xml

  ## /ansible/roles/arrs/templates/configs/arrs_config.xml.j2
  ## The config is edited in later tasks

<Config>
  <BindAddress>*</BindAddress>
  <Port></Port>
  <SslPort>9898</SslPort>
  <EnableSsl>False</EnableSsl>
  <LaunchBrowser>True</LaunchBrowser>
  <ApiKey></ApiKey>
  <AuthenticationMethod>Forms</AuthenticationMethod>
  <AuthenticationRequired>Enabled</AuthenticationRequired>
  <Branch>nightly</Branch>
  <LogLevel>info</LogLevel>
  <SslCertPath></SslCertPath>
  <SslCertPassword></SslCertPassword>
  <UrlBase></UrlBase>
  <InstanceName></InstanceName>
  <UpdateMechanism>Docker</UpdateMechanism>
</Config>

```

Bazarr has its own config and is templated with desired variables here:

```yaml

  ## /ansible/roles/arrs/templates/configs/bazarr_config.yaml.j2

---
addic7ed:
  cookies: ''
  password: ''
  user_agent: ''
  username: ''
  vip: false
analytics:
  enabled: true
anidb:
  api_client: ''
  api_client_ver: 1
animetosho:
  anidb_api_client: ''
  anidb_api_client_ver: 1
  search_threshold: 6
anticaptcha:
  anti_captcha_key: ''
assrt:
  token: ''
auth:
  apikey: {{ bazarr_api }}
  password: ''
  type: null
  username: ''
avistaz:
  cookies: ''
  user_agent: ''
backup:
  day: 6
  folder: /config/backup
  frequency: Weekly
  hour: 3
  retention: 31
betaseries:
  token: ''
cinemaz:
  cookies: ''
  user_agent: ''
cors:
  enabled: false
deathbycaptcha:
  password: ''
  username: ''
embeddedsubtitles:
  fallback_lang: en
  hi_fallback: false
  included_codecs: []
  timeout: 600
  unknown_as_fallback: false
general:
  adaptive_searching: true
  adaptive_searching_delay: 3w
  adaptive_searching_delta: 1w
  anti_captcha_provider: null
  auto_update: true
  base_url: ''
  branch: development
  chmod: '0640'
  chmod_enabled: false
  days_to_upgrade_subs: 30
  debug: false
  default_und_audio_lang: ''
  default_und_embedded_subtitles_lang: ''
  dont_notify_manual_actions: false
  embedded_subs_show_desired: true
  embedded_subtitles_parser: ffprobe
  enabled_integrations: []
  enabled_providers:
  - opensubtitlescom
  - embeddedsubtitles
  flask_secret_key: {{ bazarr_flask_key }}
  hi_extension: hi
  ignore_ass_subs: false
  ignore_pgs_subs: false
  ignore_vobsub_subs: false
  ip: 0.0.0.0
  language_equals: []
  minimum_score: 90
  minimum_score_movie: 70
  movie_default_enabled: true
  movie_default_profile: 1
  multithreading: true
  page_size: 25
  parse_embedded_audio_track: false
  path_mappings: []
  path_mappings_movie: []
  port: {{ bazarr_ports_cont }}
  postprocessing_cmd: ''
  postprocessing_threshold: 90
  postprocessing_threshold_movie: 70
  serie_default_enabled: true
  serie_default_profile: 1
  single_language: false
  skip_hashing: false
  subfolder: current
  subfolder_custom: ''
  subzero_mods: OCR_fixes,remove_tags,remove_HI,common,fix_uppercase
  theme: auto
  upgrade_frequency: 12
  upgrade_manual: true
  upgrade_subs: true
  use_embedded_subs: true
  use_postprocessing: false
  use_postprocessing_threshold: false
  use_postprocessing_threshold_movie: false
  use_radarr: true
  use_scenename: true
  use_sonarr: true
  utf8_encode: true
  wanted_search_frequency: 6
  wanted_search_frequency_movie: 6
hdbits:
  passkey: ''
  username: ''
karagarga:
  f_password: ''
  f_username: ''
  password: ''
  username: ''
ktuvit:
  email: ''
  hashed_password: ''
legendasdivx:
  password: ''
  skip_wrong_fps: false
  username: ''
log:
  exclude_filter: ''
  ignore_case: false
  include_filter: ''
  use_regex: false
movie_scores:
  audio_codec: 3
  edition: 1
  hash: 119
  hearing_impaired: 1
  release_group: 13
  resolution: 2
  source: 7
  streaming_service: 1
  title: 60
  video_codec: 2
  year: 30
napisy24:
  password: ''
  username: ''
opensubtitles:
  password: ''
  skip_wrong_fps: false
  ssl: false
  timeout: 15
  use_tag_search: false
  username: ''
  vip: false
opensubtitlescom:
  include_ai_translated: false
  password: {{ bazarr_opensub_pass }}
  use_hash: true
  username: {{ bazarr_opensub_user }}
podnapisi:
  verify_ssl: true
postgresql:
  database: bazarr
  enabled: true
  host: {{ postgres_name }}
  password: {{ postgres_password }}
  port: {{ postgres_ports_cont }}
  username: {{ postgres_username }}
proxy:
  exclude:
  - localhost
  - 127.0.0.1
  password: ''
  port: ''
  type: null
  url: ''
  username: ''
radarr:
  apikey: {{ radarr_api }}
  base_url: ''
  defer_search_signalr: false
  excluded_tags: []
  full_update: Daily
  full_update_day: 6
  full_update_hour: 4
  http_timeout: 60
  ip: {{ radarr_name }}
  movies_sync: 60
  only_monitored: false
  port: {{ radarr_ports_cont }}
  ssl: false
  sync_only_monitored_movies: false
  use_ffprobe_cache: true
series_scores:
  audio_codec: 3
  episode: 30
  hash: 359
  hearing_impaired: 1
  release_group: 14
  resolution: 2
  season: 30
  series: 180
  source: 7
  streaming_service: 1
  video_codec: 2
  year: 90
sonarr:
  apikey: {{ sonarr_api }}
  base_url: ''
  defer_search_signalr: false
  exclude_season_zero: false
  excluded_series_types: []
  excluded_tags: []
  full_update: Daily
  full_update_day: 6
  full_update_hour: 4
  http_timeout: 60
  ip: {{ sonarr_name }}
  only_monitored: false
  port: {{ sonarr_ports_cont }}
  series_sync: 60
  ssl: false
  sync_only_monitored_episodes: false
  sync_only_monitored_series: false
  use_ffprobe_cache: true
subf2m:
  user_agent: ''
  verify_ssl: true
subsync:
  checker:
    blacklisted_languages: []
    blacklisted_providers: []
  debug: false
  force_audio: false
  gss: true
  max_offset_seconds: 60
  no_fix_framerate: true
  subsync_movie_threshold: 70
  subsync_threshold: 90
  use_subsync: false
  use_subsync_movie_threshold: false
  use_subsync_threshold: false
titlovi:
  password: ''
  username: ''
titulky:
  approved_only: false
  password: ''
  username: ''
whisperai:
  endpoint: http://127.0.0.1:9000
  loglevel: INFO
  response: 5
  timeout: 3600
xsubs:
  password: ''
  username: ''

```

***

### Configs

***

Next, I include config sub-tasks to edit the arrs config for each service:

```yaml

  ## /ansible/roles/arrs/tasks/main.yml

################################
# CONFIGS
################################

- name: Conduct config tasks
  ansible.builtin.include_tasks: sub_tasks/config.yml
  vars:
    config_api: '{{ item.api }}'
    config_location: '{{ item.location }}'
    config_name: '{{ item.name }}'
    config_port: '{{ item.port }}'
  loop:
    - { api: '{{ lidarr_api }}', location: '{{ lidarr_location }}', name: '{{ lidarr_name }}', port: '{{ lidarr_ports_cont }}' }
    - { api: '{{ prowlarr_api }}', location: '{{ prowlarr_location }}', name: '{{ prowlarr_name }}', port: '{{ prowlarr_ports_cont }}' }
    - { api: '{{ radarr_api }}', location: '{{ radarr_location }}', name: '{{ radarr_name }}', port: '{{ radarr_ports_cont }}' }
    - { api: '{{ radarr_4k_api }}', location: '{{ radarr_4k_location }}', name: '{{ radarr_4k_name }}', port: '{{ radarr_4k_ports_cont }}' }
    - { api: '{{ sonarr_api }}', location: '{{ sonarr_location }}', name: '{{ sonarr_name }}', port: '{{ sonarr_ports_cont }}' }
    - { api: '{{ sonarr_4k_api }}', location: '{{ sonarr_4k_location }}', name: '{{ sonarr_4k_name }}', port: '{{ sonarr_4k_ports_cont }}' }
    - { api: '{{ whisparr_api }}', location: '{{ whisparr_location }}', name: '{{ whisparr_name }}', port: '{{ whisparr_ports_cont }}' }

```

These sub-tasks set the API, instance name and port for each arrs config:

```yaml

  ## /ansible/roles/arrs/tasks/sub_tasks/config.yml

- name: Conduct config tasks
  block:
    - name: Lookup Apikey value
      community.general.xml:
        path: '{{ config_location }}/config.xml'
        xpath: /Config/ApiKey
        content: text
      register: arrs_xml_api

    - name: Insert existing api key
      when: (arrs_xml_api.matches[0].ApiKey is defined) and (arrs_xml_api.matches[0].ApiKey != config_api)
      community.general.xml:
        path: '{{ config_location }}/config.xml'
        xpath: /Config/ApiKey
        value: '{{ config_api }}'

    - name: Lookup Port value
      community.general.xml:
        path: '{{ config_location }}/config.xml'
        xpath: /Config/Port
        content: text
      register: arrs_xml_port

    - name: Insert Port
      when: (arrs_xml_port.matches[0].Port is defined) and (arrs_xml_port.matches[0].Port != config_port)
      community.general.xml:
        path: '{{ config_location }}/config.xml'
        xpath: /Config/Port
        value: '{{ config_port }}'

    - name: Lookup InstanceName value
      community.general.xml:
        path: '{{ config_location }}/config.xml'
        xpath: /Config/InstanceName
        content: text
      register: arrs_xml_instance

    - name: Insert InstanceName
      when: (arrs_xml_instance.matches[0].InstanceName is defined) and (arrs_xml_instance.matches[0].InstanceName != config_name)
      community.general.xml:
        path: '{{ config_location }}/config.xml'
        xpath: /Config/InstanceName
        value: '{{ config_name }}'

    - name: Lookup AuthenticationMethod value
      community.general.xml:
        path: '{{ config_location }}/config.xml'
        xpath: /Config/AuthenticationMethod
        content: text
      register: arrs_xml_external

    - name: Change auth method to external
      when: ((arrs_xml_external.matches[0].AuthenticationMethod is defined) and (arrs_xml_external.matches[0].AuthenticationMethod != 'External'))
      community.general.xml:
        path: '{{ config_location }}/config.xml'
        xpath: /Config/AuthenticationMethod
        value: External

```

Additionally, I set the auth method to external. Note:
   - I protect each instance with Authelia, making the built-in login redundant
   - Do not do this if exposing services (i.e, via reverse proxy) without a SSO provider, or some other means to prevent unwanted access.

***

### Postgres

***

I opt for Postgres databases for all my arrs services.

The first step is to include the postgres common tasks:

([Click for overview](https://drjoyce.blog/common/#postgres))

```yaml

  ## /ansible/roles/arrs/tasks/main.yml

################################
# POSTGRES (DATABASE)
################################

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
    - 'whisparr-main-v3'
    - 'whisparr-log-v3'

```

I then edit the configs with relevant Postgres variables:

```yaml

  ## /ansible/roles/arrs/tasks/main.yml

################################
# POSTGRES (CONFIG)
################################

- name: Conduct Postgres config tasks
  ansible.builtin.include_tasks: sub_tasks/postgres.yml
  vars:
    arrs_config_location: '{{ item.location }}'
    arrs_log_db: '{{ item.logdb }}'
    arrs_main_db: '{{ item.maindb }}'
  loop:
    - { location: '{{ lidarr_location }}', logdb: 'lidarr-log', maindb: 'lidarr-main' }
    - { location: '{{ prowlarr_location }}', logdb: 'prowlarr-log', maindb: 'prowlarr-main' }
    - { location: '{{ radarr_location }}', logdb: 'radarr-log', maindb: 'radarr-main' }
    - { location: '{{ radarr_4k_location }}', logdb: 'radarr-4k-log', maindb: 'radarr-4k-main' }
    - { location: '{{ sonarr_location }}', logdb: 'sonarr-log', maindb: 'sonarr-main', cachedb: '' }
    - { location: '{{ sonarr_4k_location }}', logdb: 'sonarr-4k-log', maindb: 'sonarr-4k-main' }
    - { location: '{{ whisparr_location }}', logdb: 'whisparr-log-v3', maindb: 'whisparr-main-v3' }

```

```yaml

  ## /ansible/roles/arrs/tasks/sub_tasks/postgres.yml

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

```

Lastly, I include tasks to remove any sqlite databases:

```yaml

  ## /ansible/roles/arrs/tasks/main.yml

################################
# POSTGRES (CLEAN UP)
################################

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
    - { location: '{{ sonarr_location }}', name: 'sonarr', backups: '{{ sonarr_location }}/Backups' }
    - { location: '{{ sonarr_4k_location }}', name: 'sonarr', backups: '{{ sonarr_4k_location }}/Backups' }
    - { location: '{{ whisparr_location }}', name: 'whisparr', backups: '{{ whisparr_location }}/Backups' }

```

```yaml

  ## /ansible/roles/arrs/tasks/sub_tasks/sqlite.yml

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

**Note:** I'm okay with deleting sqlite databases as I've already migrated to Postgres

***

### Cloudflare DNS

***

To prepare for reverse proxy access, I create DNS records for each service:

([Click for overview](https://drjoyce.blog/common/#cloudflare-dns))

```yaml

  ## /ansible/roles/arrs/tasks/main.yml

################################
# CLOUDFLARE
################################

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
    - '{{ bazarr_name }}'
    - '{{ lidarr_name }}'
    - '{{ prowlarr_name }}'
    - '{{ radarr_name }}'
    - '{{ radarr_4k_name }}'
    - '{{ sonarr_name }}'
    - '{{ sonarr_4k_name }}'
    - '{{ whisparr_name }}'

```

***

### Traefik Labels

***

Next, I form the Traefik (reverse-proxy) labels for each service:

([Click for overview](https://drjoyce.blog/common/#traefik-labels))

```yaml

  ## /ansible/roles/arrs/tasks/main.yml

################################
# TRAEFIK
################################

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
    ## prowlarr
    - { var: 'prowlarr_labels', 
        name: '{{ prowlarr_name }}', 
        port: '{{ prowlarr_ports_cont }}', 
        api: 'PathRegexp(`/[0-9]+/api`) || PathRegexp(`/[0-9]+/download`) || PathPrefix(`/api`) || PathPrefix(`/ping`)', 
        sso: ',authelia@swarm', 
        tp: ',themepark-prowlarr', 
        tp_app: 'prowlarr' }
    ## radarr
    - { var: 'radarr_labels', 
        name: '{{ radarr_name }}', 
        port: '{{ radarr_ports_cont }}', 
        api: 'PathPrefix(`/api`) || PathPrefix(`/feed`) || PathPrefix(`/ping`)', 
        sso: ',authelia@swarm', 
        tp: ',themepark-radarr', 
        tp_app: 'radarr' }
    ## radarr-4k
    - { var: 'radarr_4k_labels', 
        name: '{{ radarr_4k_name }}', 
        port: '{{ radarr_4k_ports_cont }}', 
        api: 'PathPrefix(`/api`) || PathPrefix(`/feed`) || PathPrefix(`/ping`)', 
        sso: ',authelia@swarm', 
        tp: ',themepark-radarr4k', 
        tp_app: 'radarr' }
    ## sonarr
    - { var: 'sonarr_labels', 
        name: '{{ sonarr_name }}', 
        port: '{{ sonarr_ports_cont }}', 
        api: 'PathPrefix(`/api`) || PathPrefix(`/feed`) || PathPrefix(`/ping`)', 
        sso: ',authelia@swarm', 
        tp: ',themepark-sonarr', 
        tp_app: 'sonarr' }
    ## sonarr-4k
    - { var: 'sonarr_4k_labels', 
        name: '{{ sonarr_4k_name }}', 
        port: '{{ sonarr_4k_ports_cont }}', 
        api: 'PathPrefix(`/api`) || PathPrefix(`/feed`) || PathPrefix(`/ping`)', 
        sso: ',authelia@swarm', 
        tp: ',themepark-sonarr4k', 
        tp_app: 'sonarr' }
    ## whisparr
    - { var: 'whisparr_labels', 
        name: '{{ whisparr_name }}', 
        port: '{{ whisparr_ports_cont }}', 
        api: 'PathPrefix(`/api`) || PathPrefix(`/feed`) || PathPrefix(`/ping`)', 
        sso: ',authelia@swarm', 
        tp: ',themepark-whisparr', 
        tp_app: 'whisparr' }

```

***

### Stack Deploy

***

Lastly, with everything in order, it's time to deploy the stack:

([Click for overview](https://drjoyce.blog/common/#stacks))

```yaml

  ## /ansible/roles/arrs/tasks/main.yml

################################
# DEPLOY
################################

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

```yaml

  ## /ansible/roles/arrs/templates/arrs-stack.yml.j2

services:
  {{ bazarr_name }}:
    image: {{ bazarr_image_repo }}:{{ bazarr_image_tag }}
    networks:
      - {{ network_overlay }}
    environment:
      PUID: '{{ puid }}'
      PGID: '{{ pgid }}'
      TZ: '{{ timezone }}'
      WEBUI_PORTS: '{{ bazarr_ports_cont }}/tcp,{{ bazarr_ports_cont }}/udp'
      TP_SCHEME: 'http'
      TP_DOMAIN: '{{ themepark_domain }}'
      TP_HOTIO: 'true'
      TP_THEME: '{{ themepark_theme }}'
    ports:
      - target: {{ bazarr_ports_cont }}
        published: {{ bazarr_ports_host }}
        protocol: tcp
        mode: ingress
    volumes:
      - type: bind
        source: {{ bazarr_location }}
        target: /config
      - type: bind
        source: {{ themes_location }}/98-themepark-bazarr
        target: /etc/cont-init.d/98-themepark
      - type: bind
        source: /mnt
        target: /mnt
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:{{ bazarr_ports_cont }}/login"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 10s
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.labels.ansible_host == localhost]
      labels: {{ bazarr_labels }}
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s

  {{ lidarr_name }}:
    image: {{ lidarr_image_repo }}:{{ lidarr_image_tag }}
    networks:
      - {{ network_overlay }}
    environment:
      PUID: '{{ puid }}'
      PGID: '{{ pgid }}'
      TZ: '{{ timezone }}'
      TP_SCHEME: 'http'
      TP_DOMAIN: '{{ themepark_domain }}'
      TP_HOTIO: 'true'
      TP_THEME: '{{ themepark_theme }}'
    ports:
      - target: {{ lidarr_ports_cont }}
        published: {{ lidarr_ports_host }}
        protocol: tcp
        mode: ingress
    volumes:
      - type: bind
        source: {{ lidarr_location }}
        target: /config
      - type: bind
        source: {{ themes_location }}/98-themepark-lidarr
        target: /etc/cont-init.d/98-themepark
      - type: bind
        source: /mnt
        target: /mnt
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:{{ lidarr_ports_cont }}/login"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 10s
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.labels.ansible_host == localhost]
      labels: {{ lidarr_labels }}
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s

  {{ prowlarr_name }}:
    image: {{ prowlarr_image_repo }}:{{ prowlarr_image_tag }}
    networks:
      - {{ network_overlay }}
    environment:
      PUID: '{{ puid }}'
      PGID: '{{ pgid }}'
      TZ: '{{ timezone }}'
      TP_SCHEME: 'http'
      TP_DOMAIN: '{{ themepark_domain }}'
      TP_HOTIO: 'true'
      TP_THEME: '{{ themepark_theme }}'
    ports:
      - target: {{ prowlarr_ports_cont }}
        published: {{ prowlarr_ports_host }}
        protocol: tcp
        mode: ingress
    volumes:
      - type: bind
        source: {{ prowlarr_location }}
        target: /config
      - type: bind
        source: {{ themes_location }}/98-themepark-prowlarr
        target: /etc/cont-init.d/98-themepark
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:{{ prowlarr_ports_cont }}/login"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 10s
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.labels.ansible_host == localhost]
      labels: {{ prowlarr_labels }}
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s

  {{ radarr_name }}:
    image: {{ radarr_image_repo }}:{{ radarr_image_tag }}
    networks:
      - {{ network_overlay }}
    environment:
      PUID: '{{ puid }}'
      PGID: '{{ pgid }}'
      TZ: '{{ timezone }}'
      TP_SCHEME: 'http'
      TP_DOMAIN: '{{ themepark_domain }}'
      TP_HOTIO: 'true'
      TP_THEME: '{{ themepark_theme }}'
    ports:
      - target: {{ radarr_ports_cont }}
        published: {{ radarr_ports_host }}
        protocol: tcp
        mode: ingress
    volumes:
      - type: bind
        source: {{ radarr_location }}
        target: /config
      - type: bind
        source: {{ themes_location }}/98-themepark-radarr
        target: /etc/cont-init.d/98-themepark
      - type: bind
        source: /mnt
        target: /mnt
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:{{ radarr_ports_cont }}/login"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 10s
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.labels.ansible_host == localhost]
      labels: {{ radarr_labels }}
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s

  {{ radarr_4k_name }}:
    image: {{ radarr_4k_image_repo }}:{{ radarr_4k_image_tag }}
    networks:
      - {{ network_overlay }}
    environment:
      PUID: '{{ puid }}'
      PGID: '{{ pgid }}'
      TZ: '{{ timezone }}'
      TP_SCHEME: 'http'
      TP_DOMAIN: '{{ themepark_domain }}'
      TP_HOTIO: 'true'
      TP_THEME: '{{ themepark_theme }}'
    ports:
      - target: {{ radarr_4k_ports_cont }}
        published: {{ radarr_4k_ports_host }}
        protocol: tcp
        mode: ingress
    volumes:
      - type: bind
        source: {{ radarr_4k_location }}
        target: /config
      - type: bind
        source: {{ themes_location }}/98-themepark-radarr
        target: /etc/cont-init.d/98-themepark
      - type: bind
        source: /mnt
        target: /mnt
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:{{ radarr_4k_ports_cont }}/login"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 10s
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.labels.ansible_host == localhost]
      labels: {{ radarr_4k_labels }}
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s

  {{ sonarr_name }}:
    image: {{ sonarr_image_repo }}:{{ sonarr_image_tag }}
    networks:
      - {{ network_overlay }}
    environment:
      PUID: '{{ puid }}'
      PGID: '{{ pgid }}'
      TZ: '{{ timezone }}'
      TP_SCHEME: 'http'
      TP_DOMAIN: '{{ themepark_domain }}'
      TP_HOTIO: 'true'
      TP_THEME: '{{ themepark_theme }}'
    ports:
      - target: {{ sonarr_ports_cont }}
        published: {{ sonarr_ports_host }}
        protocol: tcp
        mode: ingress
    volumes:
      - type: bind
        source: {{ sonarr_location }}
        target: /config
      - type: bind
        source: {{ themes_location }}/98-themepark-sonarr
        target: /etc/cont-init.d/98-themepark
      - type: bind
        source: /mnt
        target: /mnt
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:{{ sonarr_ports_cont }}/login"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 10s
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.labels.ansible_host == localhost]
      labels: {{ sonarr_labels }}
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s

  {{ sonarr_4k_name }}:
    image: {{ sonarr_4k_image_repo }}:{{ sonarr_4k_image_tag }}
    networks:
      - {{ network_overlay }}
    environment:
      PUID: '{{ puid }}'
      PGID: '{{ pgid }}'
      TZ: '{{ timezone }}'
      TP_SCHEME: 'http'
      TP_DOMAIN: '{{ themepark_domain }}'
      TP_HOTIO: 'true'
      TP_THEME: '{{ themepark_theme }}'
    ports:
      - target: {{ sonarr_4k_ports_cont }}
        published: {{ sonarr_4k_ports_host }}
        protocol: tcp
        mode: ingress
    volumes:
      - type: bind
        source: {{ sonarr_4k_location }}
        target: /config
      - type: bind
        source: {{ themes_location }}/98-themepark-sonarr
        target: /etc/cont-init.d/98-themepark
      - type: bind
        source: /mnt
        target: /mnt
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:{{ sonarr_4k_ports_cont }}/login"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 10s
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.labels.ansible_host == localhost]
      labels: {{ sonarr_4k_labels }}
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s

  {{ whisparr_name }}:
    image: {{ whisparr_image_repo }}:{{ whisparr_image_tag }}
    networks:
      - {{ network_overlay }}
    environment:
      PUID: '{{ puid }}'
      PGID: '{{ pgid }}'
      TZ: '{{ timezone }}'
      TP_SCHEME: 'http'
      TP_DOMAIN: '{{ themepark_domain }}'
      TP_HOTIO: 'true'
      TP_THEME: '{{ themepark_theme }}'
    ports:
      - target: {{ whisparr_ports_cont }}
        published: {{ whisparr_ports_host }}
        protocol: tcp
        mode: ingress
    volumes:
      - type: bind
        source: {{ whisparr_location }}
        target: /config
      - type: bind
        source: {{ themes_location }}/98-themepark-whisparr
        target: /etc/cont-init.d/98-themepark
      - type: bind
        source: /mnt
        target: /mnt
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:{{ whisparr_ports_cont }}/login"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 10s
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.labels.ansible_host == localhost]
      labels: {{ whisparr_labels }}
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s

networks:
  {{ network_overlay }}:
    external: true

```

Whether you decide to deploy a compose file is up to you. [See here](https://drjoyce.blog/roles/#to-compose-or-not-to-compose).


<div align=center>
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="lebowski89" data-color="#FFDD00" data-emoji=""  data-font="Lato" data-text="Buy me a coffee" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>
</div>

<script data-name="BMC-Widget" data-cfasync="false" src="https://cdnjs.buymeacoffee.com/1.0.0/widget.prod.min.js" data-id="lebowski89" data-description="Support me on Buy me a coffee!" data-message="Support Me" data-color="#5F7FFF" data-position="Right" data-x_margin="18" data-y_margin="18"></script>