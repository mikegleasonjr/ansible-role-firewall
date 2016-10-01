Ansible Firewall Role
=========

[![Build Status](https://travis-ci.org/mikegleasonjr/ansible-role-firewall.svg?branch=master)](https://travis-ci.org/mikegleasonjr/ansible-role-firewall)
[![Ansible Galaxy](https://img.shields.io/badge/galaxy-mikegleasonjr.firewall-5bbdbf.svg?style=flat)](https://galaxy.ansible.com/detail#/role/5878)

After I found out `UFW` was too limited in terms of functionalities, I tried several firewall roles out there but none satisfied the requirements I had:

- Support virtually all iptables rules from the start
- Allow granular rules addition/overriding for specific hosts
- Easily inject variables in the rules
- Allow rules ordering
- Simplicity (not having to learn how role variables would generate the rules)
- Persistence (reload the rules at boot)

This role is an attempt to solve these requirements. It currently supports only ipv4 on Debian and RedHat distributions.

Requirements
------------

`iptables` (installed by default on all official Debian and RedHat distributions)

Installation
------------

`$ ansible-galaxy install mikegleasonjr.firewall`

Role Variables
--------------

There are only 3 dictionaries to override in `defaults/main.yml`:

```
firewall_v4_default_rules:
  001 default policies:
    - -P INPUT ACCEPT
    - -P OUTPUT ACCEPT
    - -P FORWARD DROP
  002 allow loopback:
    - -A INPUT -i lo -s 127.0.0.0/8 -d 127.0.0.0/8 -j ACCEPT
  003 allow ping replies:
    - -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
    - -A OUTPUT -p icmp --icmp-type echo-reply -j ACCEPT
  100 allow established related:
    - -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
  200 allow ssh:
    - -A INPUT -p tcp --dport ssh -j ACCEPT
  999 drop everything:
    - -P INPUT DROP

firewall_v4_group_rules: {}

firewall_v4_host_rules: {}

```

The keys to the dictionaries (`001 default policies`, `002 allow loopback`, ...) can be anything. They are only used for rules **ordering** and **overriding** (explained later). On rules generation, the keys are sorted alphabetically. Hence the 001s and 999s.

Those defaults will generate the following script to be executed on the host:

```
#!/bin/sh
# Ansible managed: <redacted>

# flush rules & delete user-defined chains
iptables -F
iptables -X
iptables -t raw -F
iptables -t raw -X
iptables -t nat -F
iptables -t nat -X
iptables -t mangle -F
iptables -t mangle -X

# 001 default policies
iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD DROP

# 002 allow loopback
iptables -A INPUT -i lo -s 127.0.0.0/8 -d 127.0.0.0/8 -j ACCEPT

# 003 allow ping replies
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
iptables -A OUTPUT -p icmp --icmp-type echo-reply -j ACCEPT

# 100 allow established related
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# 200 allow ssh
iptables -A INPUT -p tcp --dport ssh -j ACCEPT

# 999 drop everything
iptables -P INPUT DROP
```

As you can see, the rules are ordered by the dictionary key. You can also observe that you can do pretty much what you want with the rules. In fact, the rules defined in the variables are simply the same rules you would pass to the `iptables` command. You have complete control over the rules syntax.

`$ iptables -L -n` on the host then shows...

```
Chain INPUT (policy DROP)
target     prot opt source               destination
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 8
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:22

Chain FORWARD (policy DROP)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 0
```

Now that takes care of the default rules. What about overriding?

The role provides 2 more variables where you can define more rules. Rules defined in those variables will be merged with the default rules. In fact, rules in `firewall_v4_host_rules` will be merged with `firewall_v4_group_rules`, and then the result will be merged back with the defaults.

This allows 3 levels of rules definition and overriding. I simply chose the names to match how the variable precedence works in Ansible (`all` -> `group` -> `host`). See the example playbook below to see rules overriding in action.

Example Playbook
----------------

```
- hosts: all
  roles:
    - mikegleasonjr.firewall
```

in `group_vars/all.yml` you could define the default rules for all your hosts:

```
firewall_v4_default_rules:
  001 default policies:
    - -P INPUT ACCEPT
    - -P OUTPUT ACCEPT
    - -P FORWARD DROP
  002 allow loopback:
    - -A INPUT -i lo -j ACCEPT
  003 allow ping replies:
    - -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
    - -A OUTPUT -p icmp --icmp-type echo-reply -j ACCEPT
  100 allow established related:
    - -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
  200 allow ssh limiting brute force:
    - -I INPUT -p tcp -d {{ hostvars[inventory_hostname]['ansible_eth1']['ipv4']['address'] }} --dport 22 -m state --state NEW -m recent --set
    - -I INPUT -p tcp -d {{ hostvars[inventory_hostname]['ansible_eth1']['ipv4']['address'] }} --dport 22 -m state --state NEW -m recent --update --seconds 60 --hitcount 4 -j DROP
  999 drop everything:
    - -P INPUT DROP
```

in `group_vars/webservers.yml` you would open up port 80:

```
firewall_v4_group_rules:
  400 allow web traffic:
    - -A INPUT -p tcp --dport http -j ACCEPT
```

in `host_vars/secureweb.yml` you would want to open https as well and remove ssh logins:

```
firewall_v4_host_rules:
  400 allow web traffic:
    - -A INPUT -p tcp --dport http -j ACCEPT    # need to redefine this one as well because the whole key is overwritten
    - -A INPUT -p tcp --dport https -j ACCEPT
  200 allow ssh limiting brute force: []
```

That's right, to "delete" rules, you just assign an empty list to an existing dictionary key.

To summarize, rules in `firewall_v4_host_rules` will overwrite rules in `firewall_v4_group_rules`, and then rules in `firewall_v4_group_rules` will overwrite rules in `firewall_v4_default_rules`.

You can play with the rules and see the generated script on the host at the following location: `/etc/iptables.v4.generated`.

Dependencies
------------

none

License
-------

BSD

Contributing
-------

A vagrant environment has been provided to test the role on different distributions. Add your tests in `tests.yml` and...

```
$ vagrant up
$ vagrant provision
```

Author Information
------------------

Mike Gleason jr Couturier (mikegleasonjr@gmail.com)

Other roles from the same author:

- [swap](https://github.com/mikegleasonjr/ansible-role-swap)
