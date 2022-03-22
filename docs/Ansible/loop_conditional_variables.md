# Unitary condition for each item in a loop

> Install httpd on RedHat flavors, install apache2 on Debian flavors

```yaml
{% raw %}
- name: Install package regarding the host
  ansible.builtin.package:
    name: "{{ pkg.name }}"
    state: present
  when: pkg.when
  loop:
    - { name: httpd,   when: "{{ ansible_facts.os_family == 'RedHat' }}" }
    - { name: apache2, when: "{{ ansible_facts.os_family == 'Debian' }}" }
  loop_control:
    loop_var: pkg
{% endraw %}
```

> Or

```yaml
{% raw %}
- name: Install package regarding the host
  vars:
    - os: "{{ ansible_facts.os_family }}"
  ansible.builtin.package:
    name: "{{ pkg.name }}"
    state: present
  when: pkg.when
  loop:
    - { name: httpd,   when: "{{ os == 'RedHat' }}" }
    - { name: apache2, when: "{{ os == 'Debian' }}" }
  loop_control:
    loop_var: pkg
{% endraw %}
```

> Output with 2 RHEL hosts

```
TASK [Install package regarding the host] *******************************************************************************************************************************************************************************
ok: [mh2] => (item={'name': 'httpd', 'when': True})
skipping: [mh2] => (item={'name': 'apache2', 'when': False}) 
ok: [mh1] => (item={'name': 'httpd', 'when': True})
skipping: [mh1] => (item={'name': 'apache2', 'when': False})
```
