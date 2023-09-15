# Ansible role to install k3s from rancher

Taken heavily from

- <https://www.suse.com/c/rancher_blog/deploying-k3s-with-ansible/>

## Example Inventory (at ./inv/hosts.ini)

```ini
[master]
master1

[node]
node1

[k3s_cluster:children]
master
node

[k3s_cluster:vars]
k3s_version=v1.22.3+k3s1
ansible_user=root
systemd_dir=/etc/systemd/system
master_ip="{{ hostvars[groups['master'][0]]['ansible_host'] | default(groups['master'][0]) }}"
extra_server_args=""
extra_agent_args=""
```

## Example playbook

```yaml

---

- hosts: k3s_cluster
  gather_facts: yes
  become: yes
  roles:
    - role: ansible-role-k3s/prereq
    - role: ansible-role-k3s/download

- hosts: master
  become: yes
  roles:
    - role: ansible-role-k3s/master

- hosts: node
  become: yes
  roles:
    - role: ansible-role-k3s/node

```
