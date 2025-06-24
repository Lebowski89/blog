---
draft: false
genres:
- ansible
tags:
- common-tasks
- ansible
menus: common
weight: 1
title: Handling commonly repeated tasks in Ansible
---

When you begin constructing your Ansible tasks - roles - collections, you quickly realise that there are many tasks that you will use repeatedly with little variance. For example, if you're using Ansible to help set-up and deploy your docker containers, you'll have common tasks that create directories, template configs, set-up DNS and set Traefik labels, among many others. If you're like me, I would simply copy and paste these tasks. Sure, once you do this, Ansible will run through and automate tasks all the same. However, the more tasks that you have to repeat separately in a play, the more bloated and unreadable your playbook becomes and the maintenance upkeep these tasks will require in the long-run.

To handle commonly repeated tasks, I've found it is effective to make a common task file resource, in which you then include during plays as required. In these common task are generic variables that are looped over and replaced with relevant variables as required.

The rest of this document is dedicated to describing the common tasks I use.

{{< custom-toc >}}

## Key points:
   - The common tasks are included in a resources directory in the Ansible directory
   - Each task has generic variables that will be replaced with relevant ones during the play when the role is included using the `ansible.builtin.include_tasks` module
   - Loops and iterating over hashes is key to reducing the number of required tasks.