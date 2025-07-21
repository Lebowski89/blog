---
draft: false
genres:
- ansible
menus: arrs
weight: 1
title: Deploying an arrs stack using Ansible
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

Here I define variables for service name, image repo/tag, ports and location.

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

################################
# ARRS VAULT
################################

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

```