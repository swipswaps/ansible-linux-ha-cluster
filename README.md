# ansible-linux-ha-cluster
**Ansible deployment of Linux HA (High Availability) cluster, PoC of AP-ALB.
One non-Kubernetes, non-Pacemaker, non-DRBD, without-cloud-lock-ln way to do it
even on cheapest VPSs you could find.** This is a Proof of Concept of the
reusable Ansible role [AP Application Load Balancer](https://ap-application-load-balancer.etica.ai/)
and feature a **clusterized mode** and **cross-platform cluster** and also
implements some extra, well maintained, Ansible Roles made by 3rd party.
Check the [ASCIInema demo](#asciinema-demo).

> **About using this playbooks in production**: At very minimum, you will need to
> customize the inventory (on this version, the [hosts.yml](hosts.yml)). But
> in special for who is new to Ansible and want reusability, consider the
> [fititnt/ansible-linux-ha-cluster](https://github.com/fititnt/ansible-linux-ha-cluster)
> with the following in mind:
>
>- This is a playbooks colletion great quickstart on _how to glue_ some selected
>  reusable Ansible Roles on your own private projects.
>- While the ansible-linux-ha-cluster may demonstrate cross-platform clusters
>  working together, using this on production may require some extra work than
>  you would have if choose less underlining operational systems.
>- Some playbooks, like the [infra-wireguard.yml](infra-wireguard.yml) are
>  intentionally separated both to be used alone and also to allow replacement.
>  If you already have a VPN (via your cloud provider) you may want to
>  comment/remove this playbook. Enterprise users without one VPN may want some
>  more classic VPN solution, like IPSec.

## ASCIInema demo

[![asciicast](https://asciinema.org/a/TKWVLoK0tISoZF05Q7FYZ7CcQ.svg)](https://asciinema.org/a/TKWVLoK0tISoZF05Q7FYZ7CcQ)

> _TODO: fix AP-ALB 0.8.7-beta still requiring manual changes on CentOS 7 (fititnt, 2019-12-31 02:21)_.
>
> Note: on this play, the external role `githubixx.ansible_role_wireguard` used
> on the cross-platform demo failed on Centos 8 (but weeks ago worked fine out
> of the box)

When reading the source codes or watching the ASCIInema demos, the
sufix of hosts give a hint. So `rocha_basalto_freebsd12` means FreeBSD version
12, `ap_foxtrot_debian10` means Debian 10, etc. For the role versions, check
[requirements.yml](requirements.yml) file.

<!--
Demo:

    asciinema rec ansible-linux-ha-cluster-003~v0.8-7-beta --idle-time-limit 5 --title "ansible-linux-ha-cluster-003 (AP-ALB v0.8-7-beta)"

    cat main-infra.yml && sleep 4 && cat infra-wireguard.yml && sleep 4 && cat infra-consul.yml && sleep 4 && cat infra-alb.yml && sleep 6 && cat hosts.yml

    ansible-playbook -i hosts.yml main-infra.yml

    py.test ad-hoc-alb-tests/test_alb-standard-node.py -p no:pytest-ansible --hosts='ansible://cross_platform_test_2?ansible_inventory=hosts.yml'

    asciinema upload ansible-linux-ha-cluster-003~v0.8-7-beta

-->

---

## Table of Contents

<!-- TOC depthFrom:2 -->

- [ASCIInema demo](#asciinema-demo)
- [Table of Contents](#table-of-contents)
- [Usage](#usage)
    - [How to download this ansible-linux-ha-cluster to your machine](#how-to-download-this-ansible-linux-ha-cluster-to-your-machine)
    - [How to customize and use](#how-to-customize-and-use)
    - [Ad Hoc ALB](#ad-hoc-alb)
- [ansible-linux-ha-cluster explained](#ansible-linux-ha-cluster-explained)
    - [File organization](#file-organization)
    - [Responsibilities of the Ansible Roles](#responsibilities-of-the-ansible-roles)
- [Requisites](#requisites)
    - [Ansible](#ansible)
    - [Python on target hosts](#python-on-target-hosts)
    - [Number of nodes](#number-of-nodes)
        - [But can I use only 2 nodes?](#but-can-i-use-only-2-nodes)
    - [Hardware of the cluster nodes](#hardware-of-the-cluster-nodes)
    - [The internet between the nodes](#the-internet-between-the-nodes)
        - [Did I need an dedicated IPv4? Does this work on NATed IPv4?](#did-i-need-an-dedicated-ipv4-does-this-work-on-nated-ipv4)
    - [Operatinal System of the cluster nodes](#operatinal-system-of-the-cluster-nodes)
- [License](#license)

<!-- /TOC -->

---

## Usage

Assuming you already have the [Requisites](#requisites) meet, this is how you
use:

### How to download this ansible-linux-ha-cluster to your machine

```bash
git clone https://github.com/fititnt/ansible-linux-ha-cluster.git .

# This will download https://github.com/fititnt/https://github.com/fititnt/ap-application-load-balancer
# on roles/ap-application-load-balancer
ansible-galaxy install -r requirements.yml --roles-path roles/
```

> TIP: on [requirements.yml](requirements.yml) have extra information.

### How to customize and use

```bash
# Edit to your hosts files. The default values where used on Etica.AI test servers
# You can also create a new one and change `-i hosts`
vim hosts.yml

# This will run the playbooks
ansible-playbook -i hosts.yml main-infra.yml
ansible-playbook -i hosts.yml main-apps.yml
```


### Ad Hoc ALB

```bash

ansible-playbook -i hosts.yml roles/ap-application-load-balancer/ad-hoc-alb/show-configurations-syntax-validation.yml

# Tip: you may need replace 'roles/ap-application-load-balancer/' with something like '~/.ansible/roles/ap-application-load-balancer/'
#      these ad-hoc-alb scripts are stored with the role
```

## ansible-linux-ha-cluster explained

### File organization

- [hosts.yml](hosts.yml): Almost all the logic (in Ansible term, _the inventory_)
  is on this single filethe
- [main-infra.yml](main-infra.yml) call the tasks related to infrastructure
  deployment in the ideal order.
  - [infra-wireguard.yml](infra-wireguard.yml),
    [infra-consul.yml](infra-consul.yml) and [infra-alb.yml](infra-alb.yml)
    can be played individualy (or even exchanged for other solutions)
- [main-apps.yml](main-apps.yml) exist because AP-ALB is designed to be an
  Application Load Balancer, so after deployment, is more likely that you would
  change the applications than the infrastructure.

### Responsibilities of the Ansible Roles

- **Features provided by [AP Application Load Balancer Role](https://github.com/fititnt/ap-application-load-balancer)**
  - **HAproxy 2.0.x**
  - **OpenResty/NGinx with automatic HTTPS**
    - Yes, it works out-of-the-box not only with Let's Encrypt, but also with
      load balanced to obtain, renew and share keys with the frontend clusters
- **For cluster communication**
  - Uses Consul:
    - _[Ansible role for Consul clusters by Brian Shumate](https://github.com/brianshumate/ansible-consul)_
  - Maybe Etcd will be implemented eventually
- **For provide virtual private networking**
  - Uses Wireguard
    - _[Ansible role for installing WireGuard VPN. Supports Ubuntu, Debian, Archlinx and CentOS. by Robert Wimmer](https://github.com/githubixx/ansible-role-wireguard)_
  - Note: you can disable wireguard if you already have private networkg, or
    simple use another VPN solution

## Requisites

### Ansible

Follow the [https://docs.ansible.com: Installation Guide](https://docs.ansible.com/ansible/latest/installation_guide/index.html)

Windows users can [use Windows WSL (then could choose Ubuntu 18.04 LTS)](https://docs.microsoft.com/windows/wsl/install-win10).
Check this [Windows Frequently Asked Questions](https://docs.ansible.com/ansible/latest/user_guide/windows_faq.html)

**Tip: if is your first time with Ansible, this computer is likely to be own
computer and NOT the server where you want to install ALB**. One way to explain
Ansible would be _it converts YAML variables + tasks on commands to execute
(more often) on remote hosts that can be accessed over SSH_.

### Python on target hosts

> You can skip this section if you did not get error like _"The module failed
> to execute correctly, you probably need to set the interpreter"_

The only requisite (beyond be able to SSH into a target host) that Ansible
requires is some version of Python. This is fine with most Linux distributions,
except in special CentOS 8 that decided not enforce one default python version
so was up to the user choose one.

```bash

## For CentOS 8
# Python 3, command to run if you are logged on each target host
sudo dnf install python3 -y

# Python 3, if can SSH into each host, but run an Ad-Hoc command that will
# install Python for you
ansible all -m raw -a "sudo dnf install python3 -y" -i apd.etica.ai,ape.etica.ai,apf.etica.ai,apg.etica.ai

```

### Number of nodes

Rule of thumb: **3 (three) nodes**. Have a odd number (1, 3, 5, max 7) is
important for your HA cluster decide alone what to do without humam
intervention and without [risk of brain-split](https://en.wikipedia.org/wiki/Split-brain_(computing)).

Note that obviously is possible to have more than 3 nodes (or even number of
nodes) and this rule is for services that act like master (with power of
decision), not just as followers.

#### But can I use only 2 nodes?
2 nodes, without any human intervention (or _cheating_ using using a 3rd machine
to decide who is right) is likely to have lower
[nines of availability](https://en.wikipedia.org/wiki/High_availability#"Nines")
than simply have one server and don't turn off.

It does not means that a cluster would not work with only 2 nodes. But without
one odd number, if the connection between the nodes stop, or one node crash
hard without say goodbye (like when you would do a soft reboot) the other node
could enter in read-only mode.

### Hardware of the cluster nodes
This is the **bare minimum** on each node considering alreading considering
the space of operational system:

- **1 vCPU**
  - Consul is already using concervative heath checks to reduce CPU usage,
    so it could run fine on Amazon T Nano instances
  - Wireguard is CPU friendly
    - Even using 256-bit ChaCha20 still at least as fast as IPsec (ChaPoly)
- **192 MB RAM**
  - But then please add some SWAP.
    - Don't worry, this is not kubernetes that complain about Swap.
- **8 GB disk space**
  - Maybe 5 GB if you try harder

### The internet between the nodes
**Direct access**. The ideal scenario, each node can access the other node
directly. If this is not the case, you could look at "mesh networking"
implementations.

**Lower ping is better than huge speeds**. Another ideal scenario is each
cluster have it's nodes not too far away (idealy same datacenter, but some HA
components could work fine with different datacenters on same continent but
with lower ping). If ping is higher, you may have to ajust some settings, or
accept that some tasks could be slow.

#### Did I need an dedicated IPv4? Does this work on NATed IPv4?
No. Not tested, but you could just use IPv6.

For who think this comment is strange, the cheapests VPSs from places like
<https://www.serverhunter.com/> are paid by year and does not have IPv4.

### Operatinal System of the cluster nodes
The ansible-linux-ha-cluster was tested on several linux distributions and
versions. For the role AP-ALB, some OSs may not implement all the features.
Other roles made by 3rd party are not enabled when they did not support one OS.

- **Operational System (full AP-ALB features)**:
  - **Debian Family**
    - Debian 10
    - Ubuntu Server LTS 18.04
  - **RedHat Family**
    - CentOS 8, CentOS 7
    - RHEL 8, RHEL 7
- **Tested(require extra steps, like compiling OpenResty, to implement all AP-ALB features)**
  - **Arch Linux**: _lastest_
  - **BSD Family**: _FreeBSD 12_
  - **SUSE Family**: _OpenSUSE 15_

When reading the source codes or watching the ASCIInema demos, the
sufix of hosts give a hint. So `rocha_basalto_freebsd12` means FreeBSD version
12, `ap_foxtrot_debian10` means Debian 10, etc. For the role versions, check
[requirements.yml](requirements.yml) file.

## License
[![Public Domain](https://i.creativecommons.org/p/zero/1.0/88x31.png)](UNLICENSE)

To the extent possible under law, [Etica.AI](https://etica.ai/) has waived all
copyright and related or neighboring rights to this work to
[Public Domain](UNLICENSE).
