# ansible-pve-lxc

ansible-pve-lxc is an Ansible playbook and set of roles for provisioning and configuring LXC containers in a [Proxmox](https://www.proxmox.com/en/) virtual environment, and subsequently subscribing them to a [FreeIPA](https://www.freeipa.org/page/Main_Page) domain.

Functionality:

- Provisions LXC containers on one or more PVE nodes (proxmox-lxc role)
- Checks if existing containers are already subscribed to FreeIPA domain (idm-check role)
- Configures defined containers (currently Centos 8 only) with sensible defaults (common role)
- Subscribes defined containers (currently Centos 8 only) to FreeIPA domain (idm-client role)
- Secures default sshd configuration

Todo:

- Add better `failed_when` logic to ipa-client-install task
- ~~Include Ubuntu-compatible roles~~
- Include Debian-compatible roles

## Define inventory with container configurations

- This inventory is in the YAML format, but will also work in INI format

```
---

proxmox:
  hosts:
    pve.site.domain

containers:
  hosts:
    db1.site.domain:
      ansible_host: 10.0.10.125
    app1.site.domain:
      ansible_host: 10.0.10.126
```

## Populate host variables with individual container configurations

`host_vars/db1.site.domain.yml`

```
---

node: 'gold'
vmid: '110'
cores: '1'
memory: '1024'
swap: '0'
disk: '8'
onboot: true
ostemplate: 'bulk:vztmpl/ubuntu-19.10-standard_19.10-1_amd64.tar.gz'
network: '{"net0":"name=eth0,ip=10.0.10.125/24,gw=10.0.10.1,ip6=dhcp,bridge=vmbr1"}'
```

`host_vars/app1.site.domain.yml`

```
---

node: 'gold'
vmid: '111'
cores: '2'
memory: '2048'
swap: '0'
disk: '5'
onboot: true
network: '{"net0":"name=eth0,ip=10.0.10.126/24,gw=10.0.10.1,ip6=dhcp,bridge=vmbr1"}'
```

## Configure variables

- Define PVE API and LXC default variables

`group_vars/proxmox.yml`

```
---

api_host: 'pve.site.domain'
api_user: 'root@pam'

defaults:
  node: 'gold'
  storage: 'local-zfs'
  cores: '1'
  memory: '1024'
  disk: '3'
  onboot: true
  pubkey: '{{ ssh_pubkey }}'
  searchdomain: 'site.domain'
  nameserver: '1.1.1.1,8.8.8.8'
  ostemplate: 'bulk:vztmpl/centos-8-20200402_amd64.tar.gz'
  network: '{"net0":"name=eth0,ip=dhcp,ip6=dhcp,bridge=vmbr1"}'
```

- Define PVE API password, SSH public key, and IPA admin password

`group_vars/all.yml`

```
---

api_password: '<proxmox_api_password>'
ssh_pubkey: '<ssh_public_key>'
ipa_domain: 'ipa.site.domain'
ipa_realm: 'IPA.SITE.DOMAIN'
ipa_principal: 'admin'
ipa_admin_pw: '<ipa_admin_password>'
```

- **Optional** (but suggested!): Use ansible-vault to encrypt `group_vars/all.yml`

```bash
$ ansible-vault encrypt group_vars/all.yml
```

## Use ansible-pve-lxc

- **Optional**: Temporarily disable ssh host key checking

```bash
$ export ANSIBLE_HOST_KEY_CHECKING=False
```

- Run playbook

```bash
$ ansible-playbook -i inventory site.yml --ask-become-pass
```

- Run playbook while asking for the vault password interactively

```bash
$ ansible-playbook -i inventory site.yml --ask-become-pass --ask-vault-pass
```

- **Optional**: Re-enable ssh host key checking

```bash
$ unset ANSIBLE_HOST_KEY_CHECKING
```