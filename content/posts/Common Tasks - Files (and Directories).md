## Common Tasks: Files (and Directories)

One of the simplest and most common tasks within each of my roles are those involving the `ansible.builtin.file` module, specifically the creation of directories and touching of files:

```yaml
- name: Conduct file task
  ansible.builtin.file:
    path: '{{ file_path }}'
    state: '{{ file_state }}'
    force: '{{ file_force }}'
    owner: '{{ file_owner }}'
    group: '{{ file_group }}'
    mode: '{{ file_mode }}'

- name: Wait for file
  when: (file_state != 'directory') and (file_state != 'touch')
  ansible.builtin.wait_for:
    path: '{{ file_path }}'
    state: '{{ file_state }}'
```

The above includes a `wait_for` task, to assure the file is present before continuing the play.

**Include tasks:**

```yaml
- name: Create directories
  ansible.builtin.include_tasks: '/ansible/resources/file.yml'
  vars:
    file_path: '{{ item }}'
    file_state: 'directory'
    file_force: 'false'
    file_owner: '{{ puid }}'
    file_group: '{{ pgid }}'
    file_mode: '0755'
  loop:
    - '{{ hugo_cache_location }}'
    - '{{ obsidian_location }}'
    - '{{ obsidian_location }}/vault'
    - '{{ obsidian_location }}/vault/posts'
```

In some cases the desired variables will differ between files and directories:

```yaml
- name: Create directories
  ansible.builtin.include_tasks: '/ansible/resources/file.yml'
  vars:
    file_path: '{{ item.path }}'
    file_state: 'directory'
    file_force: 'false'
    file_owner: '{{ item.owner }}'
    file_group: '{{ item.group }}'
    file_mode: '0755'
  loop:
    - { path: '{{ authelia_location }}', owner: '{{ puid }}', group: '{{ pgid }}' }
    - { path: '{{ authelia_logs_location }}', owner: '{{ puid }}', group: '{{ pgid }}' }
    - { path: '{{ authelia_redis_location }}', owner: '1001', group: '1001' }
    - { path: '{{ traefik_location }}', owner: '{{ puid }}', group: '{{ pgid }}' }
    - { path: '{{ traefik_logs_location }}', owner: '{{ puid }}', group: '{{ pgid }}' }
```

In some cases it's a simple case of 'touching' a file:

```yaml
- name: Touch acme.json
  ansible.builtin.include_tasks: '/ansible/resources/file.yml'
  vars:
    file_path: '{{ traefik_location }}/acme.json'
    file_state: 'touch'
    file_force: 'false'
    file_owner: '{{ puid }}'
    file_group: '{{ pgid }}'
    file_mode: '0600'
```