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
   - Lidarr (Music)
   - Radarr (Movies)
   - Radarr-4K (4K Movies)
   - Sonarr (TV)
   - Sonarr-4K (4K TV)
   - Whisparr (Adult)

Using Ansible, I create everything required to get these services deployed for my specific needs.

This guide will: 
   - Go through my arrs role step-by-step, from clean-up to deployment.
   - Aim to provide guidance for those looking to make their own arrs role for their own needs.

{{< image src="arrs_1.jpg" alt="matrix laptop" position="center"
    style="border-radius: 4px; width:560px; height: auto" >}}