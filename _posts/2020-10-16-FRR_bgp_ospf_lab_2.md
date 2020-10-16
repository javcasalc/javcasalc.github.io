---
layout: post
title: FRR OSPF/BGP vagrant lab (2/2)
tags: frr ospf bgp vagrant ansible
---
So, in the [first part](https://javcasalc.github.io/FRR_bgp_ospf_lab/) of this lab we did show how to configure and deploy a simple Lab with Vagrant. 
In this second part, we're going to focus on the network nodes configuration using Ansible. We will create a different Ansible role for each network type:

 - CORE (ospf) router role will have its own [ansible role](https://github.com/javcasalc/FRR_Lab/tree/master/roles/frr_core)
 - ASBR (ospf/bgp) router role will have its own [ansible role](https://github.com/javcasalc/FRR_Lab/tree/master/roles/frr_asbr)
 - BGP router role will also have its own [ansible role](https://github.com/javcasalc/FRR_Lab/tree/master/roles/frr_bgp)

## Vagrant firing up ansible playbooks

As we did show in first part, Vagrant creates every network node, but also, Vagrant is able to launch the provision stage, in this case with Ansible using the command [*ansible.playbook*](https://github.com/javcasalc/FRR_Lab/blob/a021b214563a0e67bad79fb082db1bf55464a2eb/Vagrantfile#L12):
```
config.vm.define "core" do |subconfig|
  subconfig.vm.box = "centos/7"
  subconfig.vm.hostname = "core"
  subconfig.vm.network :private_network, ip: "10.0.0.9"
  subconfig.vm.provision "ansible" do |ansible|
    ansible.playbook="core.yml"
  end
end
```
The playbook [core.yml](https://github.com/javcasalc/FRR_Lab/blob/master/core.yml) it's pretty simple:
```
---
- name: FRR provision 
  hosts: all
  roles:
    - role: frr_core
```
it just apply frr_core ansible role to *all hosts*, actually only to core node as we can see in Vagrant config.


### FRR CORE role
What does a frr ospf core node look like?. In this case we will need to run these FRR services:

 - zebra config: https://github.com/javcasalc/FRR_Lab/blob/master/roles/frr_core/templates/core_zebra.conf.j2
 - vtysh config: https://github.com/javcasalc/FRR_Lab/blob/master/roles/frr_core/templates/core_vtysh.conf.j2
 - ospf multi-instance config: https://github.com/javcasalc/FRR_Lab/blob/master/roles/frr_core/templates/core_ospfd-1.conf.j2

The [tasks main.yml](https://github.com/javcasalc/FRR_Lab/blob/master/roles/frr_core/tasks/main.yml) file will include all tasks that ansible must execute in this node. 
We can find tasks to install FRR from offical repo:
```
- name: Set DNS Google nameserver
  shell: |
    echo "nameserver 8.8.8.8" > /etc/resolv.conf
    echo "nameserver 8.8.4.4" >> /etc/resolv.conf
  become: true

- name: FRR_interfaces packages installation
  yum:
    name:
      - python-netaddr
      - tcpdump
      - net-tools
      - traceroute
    state: present
  become: true
  check_mode: no

- name: Download Official FRR packages to /tmp
  get_url:
    url: "{{ item }}"
    dest: /tmp/
  loop:
    - https://github.com/FRRouting/frr/releases/download/frr-7.1/frr-7.1-01.el7.centos.x86_64.rpm
    - https://github.com/FRRouting/frr/releases/download/frr-7.1/frr-contrib-7.1-01.el7.centos.x86_64.rpm
    - https://github.com/FRRouting/frr/releases/download/frr-7.1/frr-pythontools-7.1-01.el7.centos.x86_64.rpm
    - https://ci1.netdef.org/artifact/LIBYANG-YANGRELEASE/shared/build-10/CentOS-7-x86_64-Packages/libyang-0.16.111-0.x86_64.rpm

- name: Update system
  yum:
    name: "*"
    state: latest
  become: true

- name: Install FRR packages
  yum:
    name:
      - /tmp/libyang-0.16.111-0.x86_64.rpm
      - /tmp/frr-7.1-01.el7.centos.x86_64.rpm
      - /tmp/frr-contrib-7.1-01.el7.centos.x86_64.rpm
      - /tmp/frr-pythontools-7.1-01.el7.centos.x86_64.rpm
    state: present
  become: true
```

we can find some tasks to tweak and configure the underlaying operating system:
```
- name: Set sysctl options
  template:
    src: 99-sysctl.conf.j2
    dest: /etc/sysctl.d/99-frr.conf
  become: true

- name: Reload sysctl files
  shell: /sbin/sysctl -p
  become: true
```

and we can find some tasks to configure FRR and its services:
```
- name: Configure frr daemons file
  tags: frr_common
  template:
    src: "{{ ansible_hostname }}_daemons.j2"
    dest: /etc/frr/daemons
  become: yes
  become_user: frr

- name: Overwrite /etc/frr/zebra.conf with updated hostname
  template:
    src: "{{ ansible_hostname }}_zebra.conf.j2"
    dest: /etc/frr/zebra.conf
  become: true

- name: Configure frr vtysh config file
  tags: frr_common
  template:
    src: "{{ ansible_hostname }}_vtysh.conf.j2"
    dest: /etc/frr/vtysh.conf
  become: yes
  become_user: frr

- name: Create /etc/frr/ospfd.conf
  template:
    src: "{{ ansible_hostname}}_ospfd.conf.j2"
    dest: /etc/frr/ospfd.conf
  become: true
  become_user: frr

- name: Create /etc/frr/ospfd-1.conf
  template:
    src: "{{ ansible_hostname }}_ospfd-1.conf.j2"
    dest: /etc/frr/ospfd-1.conf
  become: true
  become_user: frr

- name: Enable ospfd in /etc/frr/daemons
  lineinfile:
    path: /etc/frr/daemons
    regexp: '^ospfd='
    line: ospfd=yes
  become: true

- name: Enable ospfd instance 1 in /etc/frr/daemons
  lineinfile:
    path: /etc/frr/daemons
    line: ospfd_instances=1
    insertafter: '^ospfd=yes'
    create: true
    state: present
  become: true

- name: Reload systemd config
  shell: /bin/systemctl daemon-reload
  become: true

- name: Start FRR
  systemd:
    name: frr
    enabled: yes
    state: started
  become: true
```
### FRR ASBR and BGP roles
Similar to FRR CORE role, you might want to inspect the ASBR and BGP roles. You'll find they pretty similar, FRR installation, FRR configuration and some OSPF/BGP simple configurations.


## Some basic running tests after vagrant deployment

After ```vagrant up``` deploys the lab environment, we can enter in any node and do some tests. 
First off, you might want to check the status of any specific node:
```
$ vagrant status core
Current machine states:  
  
core running (virtualbox)  
  
The VM is running. To stop this VM, you can run `vagrant halt` to  
shut it down forcefully, or you can run `vagrant suspend` to simply  
suspend the virtual machine. In either case, to restart it again,  
simply run `vagrant up`.
```
Which shows that core router is running ok. 
We might want to log into core router:
```
$ vagrant ssh core 
Last login: Fri Oct 16 10:25:37 2020 from 10.0.2.2  
[vagrant@localhost ~]$ 
[vagrant@localhost ~]$ whoami  
vagrant
```
notice the *vagrant* username.

If we now want to check FRR status, we can access to the VTY console:
```
[vagrant@localhost ~]$ sudo -i vtysh  
  
Hello, this is FRRouting (version 7.1).  
Copyright 1996-2005 Kunihiro Ishiguro, et al.  
  
localhost.localdomain#
```

and here we have a classic router CLI we the typical commands:

- *show version*

```
localhost.localdomain# show version  
FRRouting 7.1 (localhost.localdomain).  
Copyright 1996-2005 Kunihiro Ishiguro, et al.  
configured with:  
'--build=x86_64-redhat-linux-gnu' '--host=x86_64-redhat-linux-gnu' '--program-prefix=' '--disable-dependency-tracking' '--prefix=/usr' '--exec-prefix=/usr' '--bindir=/usr/bin' '--sysconfdir=/etc' '--datadir=/usr/share' '--includedir=  
/usr/include' '--libdir=/usr/lib64' '--libexecdir=/usr/libexec' '--localstatedir=/var' '--sharedstatedir=/var/lib' '--mandir=/usr/share/man' '--infodir=/usr/share/info' '--sbindir=/usr/lib/frr' '--sysconfdir=/etc/frr' '--localstatedir=/v  
ar/run/frr' '--disable-static' '--disable-werror' '--enable-irdp' '--enable-multipath=256' '--enable-vtysh' '--enable-ospfclient' '--enable-ospfapi' '--enable-rtadv' '--enable-ldpd' '--enable-pimd' '--enable-pbrd' '--enable-nhrpd' '--ena  
ble-eigrpd' '--enable-babeld' '--enable-user=frr' '--enable-group=frr' '--enable-vty-group=frrvty' '--enable-fpm' '--enable-watchfrr' '--disable-bgp-vnc' '--enable-isisd' '--enable-systemd' '--disable-rpki' '--enable-bfdd' 'SPHINXBUILD=s  
phinx-build' 'build_alias=x86_64-redhat-linux-gnu' 'host_alias=x86_64-redhat-linux-gnu' 'PKG_CONFIG_PATH=:/usr/lib64/pkgconfig:/usr/share/pkgconfig'
```

- *show ip protocols*:

```
localhost.localdomain# show ip protocol  
VRF: default  
Protocol : route-map  
------------------------  
system : none  
kernel : none  
connected : none  
static : none  
rip : none  
ripng : none  
ospf : none  
ospf6 : none  
isis : none  
bgp : none  
pim : none  
eigrp : none  
nhrp : none  
hsls : none  
olsr : none  
table : none  
ldp : none  
vnc : none  
vnc-direct : none  
vnc-rn : none  
bgp-direct : none  
bgp-direct-to-nve-groups : none  
babel : none  
sharp : none  
pbr : none  
bfd : none  
openfabric : none  
wildcard : none  
any : none  
localhost.localdomain#
```

- *show ip ospf 1 interface*:

```
localhost.localdomain# sho ip ospf 1 interface  
  
OSPF Instance: 1  
  
tun0 is up  
ifindex 6, MTU 1476 bytes, BW 0 Mbit <UP,POINTOPOINT,RUNNING,NOARP>  
Internet Address 192.168.0.1/30, Area 0.0.0.0  
MTU mismatch detection: enabled  
Router ID 10.0.0.9, Network Type POINTOPOINT, Cost: 10  
Transmit Delay is 1 sec, State Point-To-Point, Priority 1  
No backup designated router on this network  
Multicast group memberships: OSPFAllRouters  
Timer intervals configured, Hello 10s, Dead 40s, Wait 40s, Retransmit 5  
Hello due in 9.840s  
Neighbor Count is 0, Adjacent neighbor count is 0  
tun1 is up  
ifindex 7, MTU 1476 bytes, BW 0 Mbit <UP,POINTOPOINT,RUNNING,NOARP>  
Internet Address 192.168.0.5/30, Area 0.0.0.0  
MTU mismatch detection: enabled  
Router ID 10.0.0.9, Network Type POINTOPOINT, Cost: 10  
Transmit Delay is 1 sec, State Point-To-Point, Priority 1  
No backup designated router on this network  
Multicast group memberships: OSPFAllRouters  
Timer intervals configured, Hello 10s, Dead 40s, Wait 40s, Retransmit 5  
Hello due in 9.840s  
Neighbor Count is 0, Adjacent neighbor count is 0
```
You can check this commands in all other nodes, and check OSPF status, BGP peers status, redistribution and simulate your environment very easily.

## Summary
In the FRR Lab post we have shown how to configure some basic Ansible roles to deploy a network lab environment with some routers running OSPF and some other running OSPF/BGP and redistribution. You might want to check my GitHub repository and use to configure your own environment and make simple (or complex) simulations: https://github.com/javcasalc/FRR_Lab

