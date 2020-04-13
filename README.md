
This is an Ansible cheat sheet based on this nice YouTube video list:

https://www.youtube.com/watch?v=GMzXAbT_wlk&list=PL4CwCXuy76Fe4Lll2ksYXGtupJNxpiBVV

Official documentation:

https://docs.ansible.com/

This guide has been written on Ubuntu 18.04

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

# Tooling

There seems to be an really nice VSCode extension for Ansible.

Checkout the video down this page, you'll see code completion:

https://marketplace.visualstudio.com/items?itemName=vscoss.vscode-ansible

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
It would probably be better with (but not tested):
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

The inventory is a file containing the list of managed machines.

The default inventory is in /etc/ansible/hosts.

Here is an example:

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
Same example in yaml (you have the choice):
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


## Maintaining state & configuration drift

As you will discover below, Ansible playbooks will allow you to specify an expected state of managed machines.

Playbooks express a desired state. So when a playbook is played twice, it does nothing the second time.

But because Ansible is agentless, you have to remember applying your new playbooks to old machines. 

If you forget to do so, old machines will not have the same configuration as the new ones. 

This is called configuration drift.

Salt (see https://github.com/bfreuden/salt-cheat-sheet) is a better solution to tackle configuration drift because each managed machine has an agent (called a minion) 
ensuring the managed machine is always up-to-date.

Salt is more complex than Ansible though (not to mention the minion is consuming a fair amount of RAM). So for small infrastructures Ansible is propably a better solution.

Salt has a salt-ssh module quite similar to Ansible but, as a beginner, I found Ansible easier to use.  

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
 
 Create a firstplaybook.yml file containing:
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
ansible-playbook firstplaybook.yml
```
You can have verbose output with -v:
```bash
ansible-playbook -v firstplaybook.yml
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

That setup thing is very similar to Salt grains (see https://github.com/bfreuden/salt-cheat-sheet#grains).

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

## Jinja templates

For a bit more detailed overview of Jinja templates, see:
 
https://github.com/bfreuden/salt-cheat-sheet#jinja-temlates

And more precisely the commented version of the salt state:
 
https://github.com/bfreuden/salt-cheat-sheet#jinja-states

Let's write a playbook using Jinja templates:
```yaml
---
- hosts: all
  become: yes
  vars:
    file_version: 1.0
  tasks:
    - name: install apache2
      apt: name=apache2 update_cache=yes state=present
      notify: start apache2
    - name: install index.html
      # we're using the template module (using Jinja2)
      template:
        # this is a source file next to the playbook
        src: index.html.j2
        # the result of the template will on stored on the managed machine at this location
        dest: /var/www/html/index.html
        # you can specify a file mode
        mode: 0777
  handlers:
    - name: start apache2
      service: name=apache2 enabled=yes state=started
```

Let's add a computer_location variable to our inventory:
```ini
mail.example.com computer_location=home

[webservers]
foo.example.com computer_location=home
bar.example.com computer_location=datacenter
```

And let's write this **index.j2.html** file:
```html
<html>
<center>
    <h1>This computer hostname is {{ ansible_hostname }}</h1>
    <h2>It located at {{ computer_location }}</h2>
    <h3>It is running {{ ansible_os_family }}</h3>
    <small>This file version is {{ file_version }}</small>
    {# This is a Jinja comment that will not end up in the generated file #}
</center>
</html>
```
This template is only using variables (coming from many different sources: inventory, palybook, facts)  and comments

But the Jinja language is very powerful: you can use conditions, loops, etc...

When ran for the first time, this playbook will create the index.html file. 
When ran a second time it will do nothing because the file already exists with the expected content.
But if you change the variables of the index.j2.html, it will detect a change.

## Reusing code with include

You can compose your playbooks by reusing existing smaller bricks (either other playbooks, or simply lists of tasks).

Let's imagine this **install_apache2.yml** file that is a list of tasks installing apache2:
```yaml
---
- name: install apache2
  yum: name=httpd state=present
  when: ansible_os_family == "RedHat"

- name: install apache2
  apt: name=apache2 update_cache=yes state=present
  when: ansible_os_family == "Debian"
```

And let's imagine this update_system.yml playbook updating the system:
```yaml
---
- hosts: all
  become: yes
  tasks:
    - name: update apt cache
      apt: update_cache=yes
      when: ansible_os_family == "Debian"
    - name: update yum cache
      yum: update_cache=yes
      when: ansible_os_family == "RedHat"
```

Then you can include those files in a playbook:

```yaml
---
# this will include in this playbook all plays of the update_package_cache playbook
- include: update_package_cache.yml
# remember a playbook is a list of plays (even if all examples so far only had one)
- hosts: all
  become: yes
  tasks:
    # this will include the install_apache2 list of tasks in this playbook
    - include: install_apache2.yml
```

## Role

Includes let us compose files in order to build larger playbooks but it can lead to a mess if you don't organize them properly.

Ansible is also coming with a predefined directory structure. When you're using it, you don't even have to write the include statements. 
Files automatically imported.

Roles are ways of automatically loading certain vars_files, tasks, and handlers based on a known file structure. Grouping content by roles also allows easy sharing of roles with other users.

Official documentation:

https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html

This is an example project structure:
```text
site.yml       -> master playbook
webservers.yml -> playbook for web servers
dbservers.yml  -> playbook for db servers
roles/
    common/
        tasks/     -> contains the main list of tasks to be executed by the role.
        handlers/  -> contains handlers, which may be used by this role or even anywhere outside this role.
        files/     -> contains files which can be deployed via this role.
        templates/ -> contains templates which can be deployed via this role.
        vars/      -> other variables for the role
        defaults/  -> default variables for the role
        meta/      ->  defines some meta data for this role
    webservers/
        tasks/
        defaults/
        meta/
```

So for instance, define handlers of the apache role in **roles/apache/handlers/main.yml**:
```yaml
---
- name: start apache2 on centos
  service: name=httpd enabled=yes state=started

- name: start apache2 on ubuntu
  service: name=apache2 enabled=yes state=started
```

Then define tasks of the apache role in **roles/apache/tasks/main.yml**:
**roles/apache/tasks/main.yml**:
```yaml
---
- name: install apache2 on centos
  yum: name=httpd state=present
  when: ansible_os_family == "RedHat"
  notify: start apache2 on centos

- name: install apache2 on ubuntu
  apt: name=apache2 update_cache=yes state=present
  when: ansible_os_family == "Debian"
  notify: start apache2 on ubuntu

```

And finally define the **site.yml** that is extremely concise thanks to naming conventions:
```yaml
---
- hosts: all
  become: yes
  # apply the apache role to all hosts
  roles:
    - apache
```

And as usual:
```bash
ansible-playbook site.yml
```

As this directory structure is a standard it means you can reuse roles written by other people of the Ansible community.

## Ansible Galaxy

Ansible Galaxy is a place where people share their roles so to be reused by others.

https://galaxy.ansible.com/

When you download a role, it is stored in **/etc/ansible/roles** 
but that can be configured (*roles_path*  in **/etc/ansible/ansible.cfg**)
so you don't have to use sudo to download roles.

There is a command-line tool for Ansible Galaxy:

```bash
sudo ansible-galaxy install geerlingguy.apache geerlingguy.mysql
```

Then you car refer to those roles in your playbooks:

```yaml
---
- hosts: all
  become: yes
  roles:
    # this will work on many Linux distributions, Amazon Linux...
    - geerlingguy.apache
    # will install a master/slave configuration that will work on many distros...
    - geerlingguy.mysql
```


```bash
ansible-galaxy search apache
```

# Running ad-hoc commands 

You don't need to write ansible playbooks, you can directly run ad-hoc commands.

```bash
ansible <inventory> <options>
```

Note that it would be a mistake to write scripts calling ad-hoc commands. In that case you probably need to write a playbook.

## Examples

Running a command on all machines:
```bash
ansible all -a '/bin/date'
```

That's a shortcut for the **command** module:
```bash
ansible all -m command -a '/bin/date'
```
Ping all machines:
```bash
ansible all -m ping
```

Install a package (note the -b option to *become* admin):
```bash
ansible -b all -m apt -a "name=openssl state=latest"
```
This is better illustrated with (second one we display "root"):
```bash
ansible all -a "whoami"
ansible all -b -a "whoami"
```

Install a package with yum:
```bash
ansible all -b -m yum -a "name=httpd state=present"
```

Install the latest version of apache (make sure the latest version of apache2 is installed):
```bash
ansible all -b -m apt -a "name=apache2 state=latest"
```

Uninstall apache2 with apt (make sure apache2 is uninstalled):
```bash
ansible all -b -m apt -a "name=apache2 state=absent"
```

You can do a dry run using -C (check) option. If the module does not support check-mode, the task will be skipped.

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



