---
draft: false
genres:
- ansible
tags:
- common-tasks
- traefik
- ansible
title: Traefik Labels
---

## Common Tasks: Traefik Labels

Traefik is my reverse-proxy of choice for my docker services, with much of the config taking place in Traefik docker labels. Previously, I would define the Traefik labels for each service individually in a role defaults file. Depending on the service, I have http, https, api and theme-park labels. It can get quite extensive and much of the labels are the same across services with only variables such as the router name and port differing. 

To reduce the bloat, I make use of a labels template contained in my group_vars, a tasks file to determine which labels are required and the include_tasks task with relevant variables:

*Group_Vars:*

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

*Resources task:*

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

The above task file combines the relevant labels into a single variable, that is provided to the docker compose file, based on what is inputted into the include_tasks file:

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

In the above example, both Bazarr and Lidarr will receive full Traefik labels, including being protected by the SSO provider I have set-up (Authelia), API labels, and Theme-Park. Simply leaving the API and/or Themepark variables empty will remove these labels, i.e:

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

Moving Traefik labels to this method has allowed me to eliminate unnecessary defaults files. 