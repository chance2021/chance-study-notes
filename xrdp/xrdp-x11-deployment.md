>The guide aims to demonstrate the steps to setup X11 in a Linux server/client

>Note: In below example, both server and client are using Ubuntu18.04 system.

# XRDP Installation
```
sudo apt update
sudo apt install xfce4 xfce4-goodies xorg dbus-x11 x11-xserver-utils -y
sudo apt install xrdp -y
sudo systemctl status xrdp
sudo adduser xrdp ssl-cert  

# If ufw is active
sudo ufw allow 3389

# If cannot login, you may need to install a desktop
sudo apt install ubuntu-desktop
```

# X11 Installation
## Server

1.**Install** the related package:
```
$ sudo apt-get install xserver-xorg xserver-xorg-core x11-apps x11-xserver-utils
```

2.Update **ssh_config** file:
$ sudo vi /etc/ssh/ssh_config

Update
```
#   ForwardAgent no
#   ForwardX11 no
#   ForwardX11Trusted yes
```
To (remove “#” and change “no” to “yes”)
```
ForwardAgent yes
ForwardX11 yes
ForwardX11Trusted yes
```
>Note: For some system, you may need to update below lines as well:
```
Update…
# Port 22
# Protocol 2

To (remove “#” )

Port 22
Protocol 2
```
>Also, you need to append below line in the end of the file

```
XauthLocation /usr/bin/xauth
```
3.**Restart** ssh service:
```
$ sudo systemctl restart sshd
$ sudo service ssh restart
```

4.Install **Firefox** for later testing
```
$ sudo apt-get install firefox
```
 
## Client

1.**Install** the related package:
```
$ sudo apt-get install xauth xorg openbox
```

2.Run below command:
```
export DISPLAY=:0
```

3. **Verification**:
```
$ ssh -X <userName>@<IP Address>
```
Enter **xclock**
```
$ xclock
```
>You should be able to see below windows pop out

Type **firefox**
```
$ firefox
```

>You should be able to see the firefox windows bring up to your local desktop

 
## Ansible Playbook (XRDP Installation)
```
---
- name: Install XRDP
  hosts: chancetest
  tasks:
  - block:
    - name: apt-get update
      apt:
        update_cache: yes
    - name: Install X11 related package in Ubuntu
      apt:
        pkg:
        - xfce4
        - xfce4-goodies
        - xorg
        - dbus-x11
        - x11-xserver-utils
        - ubuntu-desktop
        state: present
    - name: Install XRDP in Ubuntu
      apt:
        name: xrdp
        state: present
    - name: Add xrdp to ssl-cert group
      user:
        name: xrdp
        groups: ssl-cert
        append: yes
    - name: Start XRDP
      service:
        name: xrdp
        enabled: yes
        state: started
    become: yes
    when: ansible_facts['distribution'] == 'Ubuntu'
```
 

>Note: You may see a pop-up window for default display manager selection. For the best permform, you can choose “lightdm”.

 
# Troubleshooting
## Issue1: Cannot login the xrdp client server with AD user

The local account can login but no AD user.

### Solution:

You have 2 choices:

**Option 1**: Use "simple" as access provider instead of Group Policy

You sssd.conf should look like this
```
[sssd]
domains = mydomain.corp
config_file_version = 2
services = nss, pam

[domain/mydomain.corp]
ad_domain = mydomain.corp
... a bunch of config not related ...
access_provider = simple
```

This makes useless the GPO Policy, but you can specify which users or groups are allowed to login with this commands in the workstation: (more info)
```
realm permit user@example.com
```
or
```
realm permit -g group@example.com.
```
 

**Option 2**: Keep Using Group Policy

This is the config that works for me in Centos 8
```
[sssd]
domains = mydomain.corp
config_file_version = 2
services = nss, pam

[domain/mydomain.corp]
ad_domain = mydomain.corp
... a bunch of config not related ...
access_provider = ad
ad_gpo_access_control = enforcing
ad_gpo_map_remote_interactive = +xrdp-sesman
```

 

 


 

