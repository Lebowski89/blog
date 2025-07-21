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

Using Ansible, I do everything required to deploy these services for my specific needs.

This guide will: 
   - Go through my arrs role step-by-step, from clean-up to deployment.
   - Aim to guide those looking to make their own role for their own needs.

***

## Group_Vars

***

