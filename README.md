# Installating wordpress using Ansible
Ansible is an open-source automation tool, or platform, used in configuration management, application deployment, intraservice orchestration, and provisioning. Ansible is used for the multi-tier deployments and it models all of IT infrastructure into one deployment instead of handling each one separately. There are no agents and no custom security architecture is required to be used in the Ansible architecture.
#### Features of Ansible
- We need to run the process only on master machine and it will automatically get transferred to client servers.
- Agentless: You don’t need to install any other software or firewall ports on the client systems you want to automate. You also don’t have to set up a separate management structure.
- Secure and consistent – Since the Ansible uses SSH and Python it is very secure and the operations are flawless.

#### Ansible installation
> Note : I'm using EC2 instance to perform the task

```sh
amazon-linux-extras install ansible2 -y
```
- The configuaration file of ansible will be
```sh
/etc/ansible/ansible.cfg
```
#### Inventory File:
The Ansible inventory file defines the hosts and groups of hosts upon which commands, modules, and tasks in a playbook operate. The default location for the inventory file is /etc/ansible/hosts. You can also create project-specific inventory files in alternate locations.

I have created a custom inventory file for my project..
```sh
vim hosts
```
```sh
[group]
<IP>  ansible_user="ec2-user" ansible_port=22 ansible_ssh_private_key_file="ansible.pem"
```
> [amazon] is the group name
I've created only one client server. If we want to add more, we can add under the same group name if all are on same OS
If the server is different OS, we need to create another group.

> Note: Ansible is using SSH protocol to connect to the client servers.

There are two methods to pass commands to ansible client
1. [Adhoc method](https://docs.ansible.com/ansible/latest/user_guide/intro_adhoc.html) : Ansible ad hoc commands are written for a very specific task. It  is a one-line command that lets you perform basic tasks efficiently without writing playbooks.
The syntax of ad-hoc commands to run is as follows and see some of the ad-hoc commands as well.
```sh
$ ansible [pattern] -m [module] -a "[module options]"
```
2. [Playbook](https://docs.ansible.com/ansible/latest/user_guide/playbooks.html) : Playbooks are files consisting of your written code, and they are written in YAML format, which defines the tasks and executes them through the Ansible. Playbooks may include one or more plays. Plays defines a set of activities or tasks to be run on hosts of inventory file.
- We need to strictly follow the syntax suggested by ansible, otherwise it will throw the error. Let's see in the following, some of the suggestions made by ansible.
- Playbooks always start in a YAML file with three dashes.
- Items that start with a single dash shall be treated as list items.
- Whitespaces matters in ansible, so the items are always defined by indentation.
 
To Run the playbook, we need to execute the command shown as follows.
$ ansible-playbook
E.g.: $ ansible-playbook sample.yaml

##### _Checking Ssh Connectivity Between Master & Client With Ping Module_
```sh
# ansible -i hosts group  -m  ping 
```
> I've created a custom host/inventory file with name hosts. 
This will check mentioned command on the server defined in hosts file under the group 'group'

> -m : module name
Ansible uses modules to run instruction. Here ping is the module name

```sh
<IP> | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}
```
> How ansible code works:
  Ansible first convert the code into python, then the code is transferred to client using scp and execute the python script on the remote server. 
  Python module must be installed on the client and master server

If we have mutiple groups, we can specify the group names or we can select all the groups using 'all'
```sh
ansible -i hosts group1,group2  -m  ping 
```

This will perform the command on all the groups mentioned in the inventory file. 
```sh
ansible -i hosts all  -m  ping 
```
This will perform the command on all the groups mentioned in the inventory file. 

#### Install lamp  using playbook.
```sh
vi lampstack.yml
```
```sh
---
- name: " install wordpress"
  become: true
  hosts: group
  vars_files:
  - variables.vars
  - mysqlvariables.var
  tasks:
  - include_tasks: wordpress.yml
```
This is the main playbook
 become: true     _## the commands needs to be ran as sudo privilege_
 hosts: group     _## the commands should be run on the servers under the group 'group'_
 hosts: group     _## variable declaration. Here we have declared variables on two files variables.vars and mysqlvariables.var_
tasks:  _## the task that needs to be executed is on the playbook wordpress.yml_

> Note I used the task for lamp-stack on separate playbook and defined a call to that on the main playbook

##### Installing lamp
```sh
wordpress.yml
```
```sh
---
- name: " install apache, PHP and mariadb"
  yum:
    name:
    - httpd
    - mariadb-server
    - MySQL-python
    state: present
  register: install_status
- name: "installing PHP"
  shell: amazon-linux-extras install php7.4 -y
- name: "mariadb/Apache - restarting/enabling service"
  when: install_status.changed == true
  service:
    name: {{ item}}
    state: restarted
    enabled: true
  with_items:
  - httpd
  - mariadb
```
> The yum module will install httpd, mariadb and mysql-python (to communicate mysql with python) and store the status on the register variable "install_status"

> when: install_status.changed == true
when HTTP and mariadb is installed, that is, whenever the register variable "install_status" value is changed to true, it will perform restart of both the services. 

Now the installation of Apache mariadb PHP is completed. 

I defined some variables for mysql and apache
```sh
cat mysqlvariables.var
```
```sh
mysql_password: "password"
wordpress_database: "wordpress"
wordpress_pass: "wordpress"
wordpress_user: "wordpress"
wordpress_url: "https://wordpress.org/wordpress-5.7.2.tar.gz"
```
```sh
cat variables.vars
```
```sh
domain_name: "gigingeorge.online"
port: "80"
user_name: "apache"
group_name: "apache"
```

We need to configure all the services. 

##### configure Virtual host for apache
```sh
- name: "Configure virtual_host"
  template:
    src: httpd.conf.tmpl
    dest: "/etc/httpd/conf.d/{{domain_name}}.conf"
  register: docroot_status
- name: "Apache conf"
  template:
    src: apache.conf.tmpl
    dest: "/etc/httpd/httpd.conf"
  register: conf_status    
- name: "creating document root"
  file:
    path: "/var/www/html/{{ domain_name }}"
    state: directory
    owner: "{{ user_name }}"
    group: "{{ group_name }}"
- name : "Syntax check httpd"
  shell: httpd -t
  register: syntax_check
```
> The file named httpd.conf.tmpl is copied to the destination directory /etc/httpd/conf.d/{{domain_name}}.conf and register the status as conf_status
I used template module to copy the values that are defined as variables. 
httpd.conf is also copied to destination server

> Aapche syntax check is performed and registered on syntax_check

##### Mysql configuration
```sh
- name: "Mysql configuration"
  ignore_errors: true
  mysql_user:
    login_user: "root"
    login_password: ""
    user: "root"
    password: "{{ mysql_password }}"
    host_all: true
  register: password_status

- name: "mariadb - removing anonymous users"
  mysql_user:
    login_user: "root"
    login_password: "{{ mysql_password }}"
    user: ""
    state: absent

- name: "Creating database for wordpress"
  mysql_db:
    login_user: "root"
    login_password: "{{ mysql_password }}"
    name: "{{ wordpress_database }}"

- name: "creating wordpress mysqluser"
  mysql_user:
    login_user: "root"
    login_password: "{{ mysql_password }}"
    user: "{{ wordpress_user }}"
    password: "{{ wordpress_pass }}"
    priv: '{{ wordpress_database }}.*:ALL'
```
> Process flow:
1. During intial installation, we will login to mysql root user with null password. 
2. We need to set root password for mysql
3. Removed all anonymous user by loging in as root with new password
4. Created wordpress database using mysql_db module
5. Created wordpress user using mysql_user module, defined the password and provided full access to the database

> ignore_errors: true    _## once the mysql is installed, and root password is set, when we re-run the play, it will thow error as mysql root login can't be possible using null password. As a result the playbook will exit the process. 
To avoid that we add  ignore_errors: true, even though there is error, the playbook will continue to check to next task

##### configure wordpress
```sh
- name: " Download wordpress"
  get_url:
    url: "{{ wordpress_url }}"
    dest: /tmp/wordpress.tar

- name: "extract wordpress"
  unarchive:
    src: /tmp/wordpress.tar
    dest: /tmp/
    remote_src: true
- name: "copying wordpress files"
  copy:
    src: "/tmp/wordpress/"
    dest: "/var/www/html/{{ domain_name }}/"
    remote_src: true

- name: copy wp-config file
  template:
    src: wp-config.php.tmpl
    dest: "/var/www/html/{{ domain_name }}/wp-config.php"
    owner: "{{user_name}}"
    group: "{{ group_name }}"
- name: " cleanup"
  file:
    path: "{{ item }}"
    state: absent
  with_items:
  - /tmp/wordpress
  - /tmp/wordpress.tar
```
> Process flow:
1. Download wordpress defined in the variable name wordpress_url and saved it to a pre-defined name so that we can extract it
2. extract the content using unarchive module
remote_src: true  ## if this is not defined, by default, the src path mentioned on the unarchive will check on Master server and since it's not present, it will throw error. If remote_src is enabled, it will check on the remote server itself.
3. copy the contents from the extracted directory to our home directory. 
4. copy the wp-config file from master server "wp-config.php.tmpl" as wp-config.php on the remote server's document root so that the database informations can be defined. 
5. remove the downloaded/extracted folders

##### Restart Apache and mysql and we made changes on conf
```sh
- name: "Restart httpd"
  when : syntax_check.rc == 0 and conf_status.changed == true or docroot_status.changed == true
  service:
    name: httpd
    state: restarted
```
```sh
- name: "Restart mariadb after setting root password"
  when: password_status.changed == true
  service:
    name: mariadb
    state: restarted
    enabled: true
```
    
The entire yml file will be
```sh
---
- name: " install apache and PHP and mariadb"
  yum:
    name:
    - httpd
    - mariadb-server
    - MySQL-python
    state: present
  register: install_status
- name: "installing PHP"
  shell: amazon-linux-extras install php7.4 -y

- name: "Configure virtual_host"
  template:
    src: httpd.conf.tmpl
    dest: "/etc/httpd/conf.d/{{domain_name}}.conf"
  register: conf_status
- name: "Apache conf"
  template:
    src: apache.conf.tmpl
    dest: "/etc/httpd/httpd.conf"
  register: conf_status  

- name: "creating document root"
  file:
    path: "/var/www/html/{{ domain_name }}"
    state: directory
    owner: "{{ user_name }}"
    group: "{{ group_name }}"

- name : "Syntax check httpd"
  shell: httpd -t
  register: syntax_check

- name: "mariadb/Apache - restarting/enabling service"
  when: install_status.changed == true
  service:
    name: {{ item}}
    state: restarted
    enabled: true
  with_items:
  - httpd
  - mariadb
- name: "Mysql configuration"
  ignore_errors: true
  mysql_user:
    login_user: "root"
    login_password: ""
    user: "root"
    password: "{{ mysql_password }}"
    host_all: true
  register: password_status

- name: "mariadb - removing anonymous users"
  mysql_user:
    login_user: "root"
    login_password: "{{ mysql_password }}"
    user: ""
    state: absent

- name: "Creating database for wordpress"
  mysql_db:
    login_user: "root"
    login_password: "{{ mysql_password }}"
    name: "{{ wordpress_database }}"

- name: "creating wordpress mysqluser"
  mysql_user:
    login_user: "root"
    login_password: "{{ mysql_password }}"
    user: "{{ wordpress_user }}"
    password: "{{ wordpress_pass }}"
    priv: '{{ wordpress_database }}.*:ALL'

- name: " Creating index.php"
  copy:
    content: "<?php phpinfo() ?>"
    dest: "/var/www/html/{{ domain_name }}/test.php"
    owner: "{{user_name}}"
    group: "{{ group_name }}"
- name: " Download wordpress"
  get_url:
    url: "{{ wordpress_url }}"
    dest: /tmp/wordpress.tar

- name: "extract wordpress"
  unarchive:
    src: /tmp/wordpress.tar
    dest: /tmp/
    remote_src: true
- name: "copying wordpress files"
  copy:
    src: "/tmp/wordpress/"
    dest: "/var/www/html/{{ domain_name }}/"
    remote_src: true
    owner: "{{user_name}}"
    group: "{{ group_name }}"


- name: copy wp-config file
  template:
    src: wp-config.php.tmpl
    dest: "/var/www/html/{{ domain_name }}/wp-config.php"
    owner: "{{user_name}}"
    group: "{{ group_name }}"
- name: " cleanup"
  file: 
    path: "{{ item }}"
    state: absent
  with_items:
  - /tmp/wordpress
  - /tmp/wordpress.tar
 
- name: "Restart and enable httpd"
  when : syntax_check.rc == 0 and conf_status.changed == true 
  service:
    name: httpd
    state: restarted
    enabled: true

- name: " Restart and enable mariadb"
  when: password_status.changed == true 
  service:
    name: mariadb
    state: restarted
    enabled: true
```




------------------------------------------ _Gigin George_---------------------------------------------------------


---------------------------------------_linkedin.com/in/gigin-george-648679142 ----------------------------------


---------------------------------- gigingkallumkal@gmail.com_-----------------------------------------------------
