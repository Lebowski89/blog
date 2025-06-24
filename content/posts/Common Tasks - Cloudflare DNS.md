## Common Tasks: Cloudflare DNS

One integral time-saving task when spinning up docker services is to add and remove DNS records using the `community.general.coudflare_dns` module:

```yaml
- name: Remove existing A DNS record
  when: cloudflare_remove_existing == 'true'
  community.general.cloudflare_dns:
    api_token: '{{ cloudflare_api }}'
    zone: '{{ cloudflare_domain }}'
    state: absent
    type: '{{ cloudflare_type }}'
    record: '{{ cloudflare_record }}'

- name: Perform Cloudflare DNS tasks
  block:
    - name: Add DNS record
      community.general.cloudflare_dns:
        api_token: '{{ cloudflare_api }}'
        zone: '{{ cloudflare_domain }}'
        state: present
        solo: '{{ cloudflare_solo }}'
        proxied: '{{ cloudflare_proxy }}'
        type: '{{ cloudflare_type }}'
        value: '{{ cloudflare_value }}'
        record: '{{ cloudflare_record }}'
      register: cloudflare_record_creation_status

    - name: Tasks on success
      when: cloudflare_record_creation_status is succeeded
      block:
        - name: Set 'dns_record_print' variable
          ansible.builtin.set_fact:
            cloudflare_record_print: '{{ (cloudflare_record == cloudflare_domain) | ternary(cloudflare_domain, cloudflare_record + "." + cloudflare_domain) }}'

        - name: Display DNS record creation status
          ansible.builtin.debug:
            msg: 'DNS A Record for "{{ cloudflare_record_print }}" set to "{{ cloudflare_value }}" was added. Proxy: {{ cloudflare_proxy }}'

```

These tasks are designed to add/remove DNS records and to display the record on success. How these tasks run is dependent on the following include_task:

```yaml
- name: Add Cloudflare DNS records
  ansible.builtin.include_tasks: '/ansible/resources/cloudflare.yml'
  vars:
    cloudflare_domain: '{{ local_domain }}'
    cloudflare_record: '{{ obsidian_name }}'
    cloudflare_type: 'A'
    cloudflare_value: '{{ ipify_public_ip }}'
    cloudflare_proxy: 'false'
    cloudflare_solo: 'true'
    cloudflare_remove_existing: 'true'
```

In the above task, the variables from the common task are replaced with the relevant values required to create a DNS record for my Obsidian container running on my local server. You can create loops to handle as many services as you require in the include task:

```yaml
- name: Add 'GitHub Pages' DNS records
  ansible.builtin.include_tasks: '/ansible/resources/cloudflare.yml'
  vars:
    cloudflare_domain: '{{ github_pages_domain }}'
    cloudflare_record: '@'
    cloudflare_type: '{{ item.type }}'
    cloudflare_value: '{{ item.value }}'
    cloudflare_proxy: 'false'
    cloudflare_solo: 'false'
    cloudflare_remove_existing: 'false'
  loop:
    - { type: 'A', value: '185.199.108.153' }
    - { type: 'A', value: '185.199.109.153' }
    - { type: 'A', value: '185.199.110.153' }
    - { type: 'A', value: '185.199.111.153' }
    - { type: 'AAAA', value: '2606:50c0:8000::153' }
    - { type: 'AAAA', value: '2606:50c0:8001::153' }
    - { type: 'AAAA', value: '2606:50c0:8002::153' }
    - { type: 'AAAA', value: '2606:50c0:8003::153' }
```

Each time the task loops the variables are replaced without interfering with each other.