# Docker Sample Application deployment with Ansible on AWS EC2

In this task, we will deploy our Docker Sample Application on AWS Cloud Infrastructure using Ansible.  Building infrastructure process is managing with control node utilizing Ansible. This infrastructure has 1 control node and 1 EC2 as worker node.

First, We will create two virtual machines on AWS with RHEL distro. To use Dynamic Inventory we need to assign tags to these machines. Folowing tags will be assigned to EC2 machines.

tag: Name=ansible_control (only control node)
tag: stack=ansible_project (all nodes)
tag: enviroment=development (only control node)


When the machines are ready, we have to run some commands on Control Node of Ansible.

Updating current packages of RHEL:

$ sudo yum update -y


To use Ansible, we need Python:

$ sudo yum install -y python3 


Now we are ready to install Ansible, so We are able to install Ansible with pip3 by following command :

$ pip3 install --user ansible


Now we can check the Ansible version:

$ ansible --version


To work in a folder, we will create a folder by using :

$ mkdir ansible-task
$ cd ansible-task


We will not use default config file of Ansible. So we will create our own config file in "ansible-task" folder. We are going to create "ansible.cfg" file and copy the following lines in this file:

[defaults]
host_key_checking = False
inventory=inventory_aws_ec2.yml
interpreter_python=auto_silent
private_key_file=/home/ec2-user/<pem_file_name>.pem 
remote_user=ec2-user


To be able to manage our resources we need our own "*.pem" file, so we have to copy it from our local machine to home directory of Control Node with line below:

$ scp -i <pem-file> <pem-file> ec2-user@<public-ip of ansible_control>:/home/ec2-user

Then we have to arrange permissions to let Ansible to use our "*.pem" file. So the following will make it done:

$ chmod 400 <pem_file_name>.pem

Now we need to assign "AmazonEC2FullAccess" role by creating it as IAM role. After assigning FullAccess role to our Control Node, we need "boto3" to use Dynamic Inventory by following:

$ pip3 install --user boto3

Now, we are ready to create our Dynamic Inventory file. We have to name our inventory file as "*_aws_ec2.yml". So our file will be named as "inventory_aws_ec2.yml" in "ansible-task" folder. And we will write the codes below in our inventory file.

plugin: aws_ec2
regions:
  - "us-east-1"
filters:
  tag:stack: ansible_project
keyed_groups:
  - key: tags.Name
  - key: tags.environment
compose:
  ansible_host: public_ip_address


By following command, we will be able to see our resources:

$ ansible-inventory -i inventory_aws_ec2.yml --graph

To make sure that all our hosts are reachable with dynamic inventory, we will run various ad-hoc commands that use the ping module. :

$ ansible all -m ping --key-file "~/<pem_file_name>.pem"

We can either pull the entire project or download it as a zip and extract the app folder out to get started with from follwing link into the "ansible-task" folder.

https://github.com/docker/getting-started/tree/master/app

Until now we configured our Control Node. We are ready to create Ansible Playbook to configure and manage our Managed Node. We will name our Playbook as "ansible.yml" and write lines of codes in it to configure our Managed Node. We are going to paste following codes top of YAML file:

---
- name: configure node
  hosts: _ansible_node
  become: true

To become "root" user, we will use "become: true" and before that, we use "_ansible_node" as host name. This name comes because of Dynamic Inventory. We are able to see host names by following command:

$ ansible-inventory -i inventory_aws_ec2.yml --graph  

Note: In config file we already assigned inventory file as "inventory_aws_ec2.yml", so we are able to see inventory without using "-i inventory_aws_ec2.yml" flag in ad-hoc command.

Updating all packages in Managed Node with "yum" module:

  tasks:
    - name: update packages
      yum: 
        name: "*"
        state: latest

Removing old versions of Docker:

    - name: Uninstall old versions
      yum:
        name: "{{ item }}"
        state: removed
      loop:
        - docker
        - docker-client
        - docker-client-latest
        - docker-common
        - docker-latest
        - docker-latest-logrotate
        - docker-logrotate
        - docker-engine

Installing yum-utils:

    - name: install yum-utils
      yum:
        name: yum-utils
        state: latest

Adding Docker Repo with "get_url" module:

    - name: Add Docker repo
      get_url:
        url: https://download.docker.com/linux/centos/docker-ce.repo
        dest: /etc/yum.repos.d/docker-ce.repo


Installing Docker with "package" module:

    - name: Install Docker
      package:
        name: docker-ce
        state: latest

Installing pip:

    - name: Install pip
      package: 
        name: python3-pip
        state: present

Installing Docker sdk and docker-compose with pip module:

    - name: Install docker sdk and docker-compose
      pip:
        name: 
          - docker
          - docker-compose


adding ec2-user to docker group with "user" module:

    - name: add ec2-user to docker group
      user:
        name: ec2-user
        groups: docker
        append: yes


Staring Docker Service with "systemd" module:

    - name: start docker service
      systemd:
        name: docker
        enabled: yes
        state: started

We will copy our source files from Control Node to Managed node, in "/tmp" folder. Copying all source files of Application to Managed Node with "copy" module:

    - name: copy source files of app
      copy:
        src: app/
        dest: tmp/

Running "docker-compose.yml" file to deploy application with "docker-compose" module:

    - name: run docker-compose
      community.docker.docker_compose:
        project_src: tmp/
        state: present
 
We are going to create "docker-compose.yml" file in "/app" folder, which has source codes of Docker Sample Application. And We will copy following lines in it:

version: "3.7"

services:
  app:
    image: node:12-alpine
    command: sh -c "yarn install && yarn run dev"
    ports:
      - 3000:3000
    working_dir: /app
    volumes:
      - ./:/app
    environment:
      MYSQL_HOST: mysql
      MYSQL_USER: root
      MYSQL_PASSWORD: secret
      MYSQL_DB: todos

  mysql:
    image: mysql:5.7
    volumes:
      - todo-mysql-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: todos

volumes:
  todo-mysql-data:


After all, we are ready to deploy our sample application on Managed node. To perform this, we will use following ad-hoc command in "ansible-task" folder:

$ ansible-playbook ansible.yml

After it is done, we will be able to see our application. We can see the application :

http://<public_ip_address_managed_node>:3000