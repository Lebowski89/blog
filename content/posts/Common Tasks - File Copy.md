## Common Tasks: File Copy

A simple task to copy files from one directory to another. I typically use this to copy files that docker services require from their role folder to the services appdata directory. I prefer to template files where and when possible, but sometimes it is better to just straight copy:

```yaml
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

**Include Tasks:**

```yaml
- name: Copy Hugo Terminal Themes files
  ansible.builtin.include_tasks: '/ansible/resources/copy.yml'
  vars:
    copy_source: '{{ item.source }}'
    copy_destination: '{{ item.destination }}'
    copy_force: 'true'
    copy_owner: '{{ puid }}'
    copy_group: '{{ pgid }}'
    copy_mode: '0644'
  loop:
    - { source: '{{ role_path }}/files/favicon.png', destination: '/{{ hugo_site_name }}/static/favicon.png' }
    - { source: '{{ role_path }}/files/og-image.png', destination: '/{{ hugo_site_name }}/static/og-image.png' }
    - { source: '{{ role_path }}/files/terminal.css', destination: '/{{ hugo_site_name }}/static/terminal.css' }
```

In the above example, I copy the files required for this blogs theme to the relevant folder.