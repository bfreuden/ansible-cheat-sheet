
This is an Ansible cheat sheet.

Official documentation:

https://docs.ansible.com/

Very nice tutorial YouTube video list:

https://www.youtube.com/watch?v=GMzXAbT_wlk&list=PL4CwCXuy76Fe4Lll2ksYXGtupJNxpiBVV

This guide has been writing on Ubuntu 18.04

# Installation

Official installation guide:

https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html

Installation can be done via os packages or via Pypi

```bash
$ sudo apt update
$ sudo apt install software-properties-common
$ sudo apt-add-repository --yes --update ppa:ansible/ansible
$ sudo apt install ansible
```

On Ubuntu 18.04 the default Python is version 2 so Ansible will create a deprecation warning.

# Managed machines

It is easier (but not required) if all machines have the same user as the one controlling them.
 
Managed machines must have an SSH access, password-less sudo, and python (3).

Enable sudo NOPASSWD for your user on all (ubuntu) machines:
```bash
for server in server1 server2 server3; do \
ssh -t $USER@$server "echo '$USER ALL=(ALL) NOPASSWD: ALL' | sudo tee -a /etc/sudoers" ; \
done
```

Deploying SSH key on your freshly installed machines:
```bash
for server in server1 server2 server3; do \
ssh $USER@$server "mkdir .ssh ; chmod 700 .ssh ; echo \"`cat ~/.ssh/id_rsa.pub`\" >> .ssh/authorized_keys" ; \
done
```
It would probably better with (but not tested):
```bash
for server in server1 server2 server3; do \
ssh-copy-id -i ~/.ssh/id_rsa.pub $USER@$server ; \
done
```

# Concepts

## Control node

Ansible is agentless and is working using SSH. The control node is basically your laptop where Ansible is installed.

## Inventory

Official documentation: 

https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html

The default inventory is /etc/ansible/hosts

The inventory is a file containing a list of machines.

```ini
mail.example.com

[webservers]
foo.example.com
bar.example.com

[dbservers]
one.example.com
two.example.com
three.example.com ansible_user=administrator

# this time with a range
[webservers2]
www[01:50].example.com
```
Same in yaml:
```yaml
all:
  hosts:
    mail.example.com:
  children:
    webservers:
      hosts:
        foo.example.com:
        bar.example.com:
    dbservers:
      hosts:
        one.example.com:
        two.example.com:
        three.example.com:
          ansible_user: administrator
    webservers2:
      hosts:
        www[01:50].example.com:

```

A server can be part of many groups but there are 2 defaults groups: **all** and **ungrouped**.
The **all** group contains every host. The **ungrouped** group contains all hosts that don’t have another group aside from **all**.

With Salt we probably would have put grains on our servers and targeted them by grain.

An inventory can be dynamically generated from a script that will pull data from an external source.

## Module

Modules control the things you're automating. They can act system files, packages, etc...

Basic structure is:
```yaml
module: directive1=value directive2=value
```
There are 450 modules by default in Ansible.


## Playbook

Playbooks are yaml files describing the desired state of something.
They are human and machine readable.

**Playbooks** contains **plays**

**plays** contains **tasks**

**tasks** call **modules**
 
 **tasks** are run sequentially
 
 **Handlers** are triggered by tasks, and are run once at the end of **plays**, only if the task has made a change!
 
 Create a firstplaybook.yaml file containing:
 ```yaml
---
# a playbook is a list of plays
- name: my first play
  # hosts to be targeted
  hosts: all
  # become root
  become: yes
  # a play is a list of tasks
  tasks:
      # task have a name that is displayed by ansible during execution
    - name: show whoami
      # here we're using the shell module with the following parameters
      shell: uname -a > /tmp/results.txt
    - name: show uname
      shell: whoami >> /tmp/results.txt
 ```
Then you can run the playbook with:
```bash
ansible-playbook firstplaybook.yaml
```
You can have verbose output with -v:
```bash
ansible-playbook -v firstplaybook.yaml
```

Playbooks have different ways to alter how it runs tasks: 
```text
with_items, failed_when, until, etc...
```

## Handler

Handlers are called when a task complete, but only if the task has changed something.
For instance if you want to copy a new service configuration, you want to restart the service.
But if the copy has done nothing (because the file is already here) you don't want to restart the service for nothing.

```yaml
---
- name: my first handler
  hosts: all
  become: yes
  tasks:
    - name: install vsftpd on ubuntu
      # run apt-get update then apt install latest version of vsftpd
      apt: name=vsftpd update_cache=yes state=latest
      # if there is an error, keep going
      # this can be useful if you target ubuntu and centos machines: 
      # just do 2 tasks, one using apt, one using yum
      # but in the end you still want the handler to be called
      ignore_errors: yes
      # call this handler if the task is successful and has done something
      # if many tasks notify this handler, it is called only once
      notify: start vsftpd
  handlers:
    # tasks refer to handlers by name, so the name is important here
    - name: start vsftpd
      # we want the make sure the service will start on boot and that is is started
      service: name=vsftpd enabled=yes state=started
```

## Variables

There are different ways to specify variables:
* Playbooks
* Files
* Command-line
* Inventories
* Facts (discovered variables)
* Set as a result of a task

```yaml
---
- name: my first handler
  hosts: all
  # define variables
  vars:
    - var1: cool stuff here
    - var2: cool stuff there
  tasks:
    - name: echo suff
      # use variables with mustache syntax
      shell: echo "{{ var1 }} is var1 but var2 is {{ var2 }}" > /tmp/result.txt
```
You can display all variables created by ansible from the target machine with:
```bash
ansible server1 -m setup
```

You can filter the output:
```bash
ansible server1 -m setup -a "filter=*processor*"
```

## Registering variable, debug and conditions
But we don't get GPU information like with Salt :-(
But we can use the shell:

```bash
ansible all -m shell -a "lspci | grep ' VGA '"
```

And maybe if we can get the output of a command as a variable?
Yes!
```yaml
---
- name: use task output as variable
  hosts: all
  tasks:
    - name: get GPU info
      shell: lspci | grep ' VGA ' | grep -o NVIDIA
      ignore_errors: yes
      # save the result in a variable (it is a JSON object)
      register: maybe_nvidia
    - name: only if nvidia
      # use 'when' to define a condition
      when: maybe_nvidia.stdout == "NVIDIA"
      # in the task we'll use the stdout of the output
      # the debug module will display messages during execution
      debug: msg="this machine has an NVIDIA GPU"

      # extra:
      # set a fact to prevent from having to do .stdout
    - set_fact:
        nvidia: "{{ maybe_nvidia.stdout }}"
    - name: only if nvidia 2
      # use the new fact
      when: nvidia == "NVIDIA"
      debug: msg="this machine has an NVIDIA GPU"
```
Do something any case of failure:
```yaml
---
# imagine we run this on Ubuntu and CentOS machines
- hosts: all
  become: yes
  tasks:
    # this one will only work on CentOS
    - name: install apache2
      yum: name=httpd state=present
      ignore_errors: yes
      register: results
    # this one will only be run on Ubuntu (because it succeeded on CentOS)
    - name: install apache2
      # will only run if the previous install as failed
      when: results.failed
      apt: name=apache2 update_cache=yes state=present
```

But we should actually do this:
```yaml
---
# imagine we run this on Ubuntu and CentOS machines
- hosts: all
  become: yes
  tasks:
    - name: install apache2
      yum: name=httpd state=present
      when: ansible_os_family == "RedHat"

    - name: install apache2
      apt: name=apache2 update_cache=yes state=present
      when: ansible_os_family == "Debian"
```

## Loops

Official documentation of loops:

https://docs.ansible.com/ansible/latest/user_guide/playbooks_loops.html

Let's use a loop to install some packages:
```yaml
---
- hosts: all
  become: yes
  tasks:
    - name: install packages
      apt: name={{ item }} update_cache=yes state=present
      loop:
        - vim
        - nano
        - apache2
```

## Role

**Roles** are a special kind of playbook that are full self-contained with tasks, variables, files, etc... 

# Running ad-hoc commands 

ansible <inventory> <options>

Running a raw command on all machines:
```bash
ansible all -a '/bin/date'
```

That's a shortcut for the **command** module:
```bash
ansible all -m command -a '/bin/date'
```

```bash
ansible all -m ping
```

Note the -b option to become run the command as sudo:
```bash
ansible -b all -m apt -a "name=openssl state=latest"
```
This is better illustrated with:
```bash
ansible all -a "whoami"
ansible all -b -a "whoami"
```

Install (make sure) a package with apt:
```bash
ansible all -b -m apt -a "name=apache2 state=present"
```

Install (make sure) the latest version of a package with apt:
```bash
ansible all -b -m apt -a "name=apache2 state=latest"
```

Uninstall (make sure) a package with apt:
```bash
ansible all -b -m apt -a "name=apache2 state=absent"
```

Compared to Salt, this method will not work for CentOS machines where you have to use the yum module:
```bash
ansible all -b -m yum -a "name=httpd state=present"
```

You can do a dry run using -C (check) option. If the module does not support check-mode, the task will be skipped.

# Maintaining state

A playbooks expresses a desired state. When it is played twice, it does not do anything the second time.


# Interesting modules

command is safer than shell, shell is safer than raw.

## command

Doesn't use shell (not affected by .bashrc files for instance), can't use pipe or redirects.

This will display the file we created with the first playbook:
```bash
ansible all -a "cat /tmp/results.txt"
```

## shell

Supports pipe or redirects. Can get messed up by user settings (/etc/bash.bashrc, .bashrc).

## raw

Just sends commands over ssh. Supports redirects. Doesn't need python.

## file

This will make sure the file we created with the first playbook is deleted:
```bash
ansible all -m file -a "path=/tmp/results.txt state=absent"
```

## copy

We can copy files:
```bash
ansible all -m copy -a "src=/etc/hosts dest=/tmp/hosts"
```
If you run it the second time it will do nothing and say that nothing has changed.



