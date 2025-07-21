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

Some of my favourite services to deploy are the servarr / arrs PVR apps, including:

   - Bazarr (Subtitles)
{{< image src="bazarr_1.jpg" alt="matrix laptop" position="center"
    style="border-radius: 2px; width:640px; height: auto" >}}
   - Lidarr (Music)
{{< image src="lidarr_1.jpg" alt="matrix laptop" position="center"
    style="border-radius: 2px; width:640px; height: auto" >}}
   - Radarr (Movies) / Radarr-4K (4K Movies)
{{< image src="radarr_1.jpg" alt="matrix laptop" position="center"
    style="border-radius: 2px; width:640px; height: auto" >}}
   - Sonarr (TV) / Sonarr-4K (4K TV)
{{< image src="sonarr_1.jpg" alt="matrix laptop" position="center"
    style="border-radius: 2px; width:640px; height: auto" >}}
   - Whisparr (Adult)
{{< image src="whisparr_1.jpg" alt="matrix laptop" position="center"
    style="border-radius: 2px; width:640px; height: auto" >}}
   - Prowlarr (Indexers)
{{< image src="prowlarr_1.jpg" alt="matrix laptop" position="center"
    style="border-radius: 2px; width:640px; height: auto" >}}

Using Ansible, I create everything required to get these services deployed for my specific needs.

This guide will: 
   - Go through my arrs role step-by-step, from clean-up to deployment.
   - Aim to guide those looking to make their own role for their own needs.