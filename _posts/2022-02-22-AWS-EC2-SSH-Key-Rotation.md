---
layout: post
title: AWS EC2 SSH Key Rotation with Ansible
author: Sebin Xavi
date: 2022-02-12 11:00:00 +0800
categories: [AWS]
tags: [aws, ansible]
math: true
mermaid: true
description: AWS EC2 SSH Key Rotation with Ansible
comments: true
---

Sometimes we get the requirement to change the key-pair of AWS EC2 instances for some security reasons. In this article, we will be changing the key pair of running EC2 instances using Ansible Playbook.


## General Information

When you use standard AMIs to launch an EC2 instance, you can connect to it using remote access protocols like SSH and RDP. AWS provides asymmetric key pairs called Amazon EC2 key pairs that you can use to authenticate yourself to the EC2 instance.
For encrypting and decrypting login information, EC2 Key Pairs are used: Public Key and Private Key. Public–key cryptography encrypts any piece of data, such as a password, with a public key, and then the receiver decrypts the data with a private key.
To connect to your instance, you must first generate a key pair, identify the name of that key pair when the instance is launched, and provide information about the private key when connecting.
As a AWS security best practice, it is necessary to regularly rotate EC2 key pairs within your account. 

### Before Running the Ansible Playbook
<p align="center">
  <img width="400" height="550" src="https://i.ibb.co/ynbHcwZ/Before-Running-Playbook1.png">
</p>

### After Running the Ansible Playbook

<p align="center">
  <img width="900" height="550" src="https://i.ibb.co/30PKTQT/After-Running-Playbook1.png">
</p>


## Technology Used
- ansible - version 2.9.20

## Prerequisites
- Ansible Master Server - Linux
- Ansible version - 2.9
- SSH access to client servers
- SSH PEM key of the client servers
- SSH User with sudo privilege
- AWS Access key and Secret Key

## Features

- Ney Keypair will be generated and Old Keypair from AWS account will be removed
- No need to mention any hosts (Inventory) as we are using Dynamic Inventory function of Ansible in this playbook.
- Rotation of key pairs for multiple instances can be done in a single session.
- Free: Ansible is an open-source tool.
- Very simple to set up and use: No special coding skills are necessary to use Ansible’s playbooks.

## About the playbook

The playbook will perform following operations;
- Gather Information of the EC2 instances in which Key To Be Rotated
- Create Inventory Of EC2 With Old SSH-keyPair
- Create New SSH-Key Material
- Add New SSH Public Key to authorized_key
- Check SSH Connectivity To EC2 instance Using Newly Added Key
- Execute the Uptime command on remote servers
- Remove Old SSH Public Key and add New SSH Public Key to authorized_key
- Print Old authorized_keys file
- Print New authorized_keys file
- Rename new SSH Private Key in Local Ansible server
- Remove Old SSH public key From AWS Account
- Add New SSH public key to AWS Account

The playbook has been written below::
~~~
---
- name: "Creation of the Ansible Inventory Of EC2 Instances in which Key To Be Rotated"
  hosts: localhost
  vars_files:
    - key.vars
  tasks:

    # ---------------------------------------------------------------
    # Gather Information of the EC2 instances in which Key To Be Rotated
    # ---------------------------------------------------------------

    - name: "Fetching Details About EC2 Instance"
      ec2_instance_info:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        filters:
          "key-name": "{{ old_key }}"
          instance-state-name: [ "running"]
      register: ec2


    # ------------------------------------------------------------
    # Creating Inventory Of EC2 With Old SSH-keyPair
    # ------------------------------------------------------------
    - name: "Creating Inventory "
      add_host:
        name: "{{ item.public_ip_address }}"
        groups: "aws"
        ansible_host: "{{ item.public_ip_address }}"
        ansible_port: "{{ ssh_port }}"
        ansible_user: "{{ system_user }}"
        ansible_ssh_private_key_file: "{{ old_key }}.pem"
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
      with_items:
        - "{{ ec2.instances }}"
      no_log: true

- name: "Updating SSH-Key Material"
  hosts: aws
  become: true
  gather_facts: false
  vars_files:
    - key.vars

  tasks:

    - name: "Register current SSH Authorized_key file of the system user"
      shell: cat /home/"{{system_user}}"/.ssh/authorized_keys
      register: oldauth

    - name: "Creating New SSH-Key Material"
      delegate_to: localhost
      run_once: True
      openssh_keypair:
        path: "{{ new_key }}"
        type: rsa
        size: 4096
        state: present

    - name: "Adding New SSH-Key Material"
      authorized_key:
        user: "{{ system_user }}"
        state: present
        key: "{{ lookup('file', '{{ new_key }}.pub')  }}"


    - name: "Creating SSH Connection Command"
      set_fact:
        ssh_connection: "ssh -o StrictHostKeyChecking=no -i {{ new_key }} {{ ansible_user }}@{{ ansible_host }} -p {{ ansible_port }} 'uptime'"


    - name: "Checking Connectivity To EC2 Using Newly Added Key"
      ignore_errors: true
      delegate_to: localhost
      shell: "{{ ssh_connection }}"

    - name: "Executing the Uptime command on remote servers"
      command: "uptime"
      register: uptimeoutput
    - debug:
        var: uptimeoutput.stdout_lines

    - name: "Removing Old SSH Public Key and adding New SSH Public Key to authorized_key"
      authorized_key:
        user: "{{ system_user }}"
        state: present
        key: "{{ lookup('file', '{{ new_key }}.pub')  }}"
        exclusive: true
    

    - name: "Print Old authorized_keys file"
      debug:
        msg: "SSH Public Keys in Old authorized_keys file are '{{ oldauth.stdout }}'"


    - name: "Print New authorized_keys file"
      shell: cat /home/"{{system_user}}"/.ssh/authorized_keys
      register: newauth
    - debug:
        msg: "SSH Public Keys in New authorized_keys file are '{{ newauth.stdout }}'"


    - name: "Renaming new Private Key Locally"
      delegate_to: localhost
      run_once: True
      shell: |
        mv {{ new_key }} {{ new_key }}.pem
        chmod 400 {{ new_key }}.pem

    - name: "Removing Old SSH public key From AWS Account"
      delegate_to: localhost
      run_once: True
      ec2_key:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        name: "{{ old_key }}"
        state: absent

    - name: "Adding New SSH public key to AWS Account"
      delegate_to: localhost
      run_once: True
      ec2_key:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        name: "{{ new_key }}"
        key_material: "{{ lookup('file', '{{ new_key }}.pub') }}"
        state: present
~~~

## Setup

### Installation of Ansible and Boto (In Ubuntu)

~~~sh
$ apt-get update
$ apt-get install python3
$ apt-get install python3-pip
$ pip3 install ansible
$ pip3 install boto3 botocore
~~~
![alt text](https://i.ibb.co/bdBtdhk/1.png)

## Usage

##### 1. Pull the code from github repository

~~~
$ git clone https://github.com/sebinxavi/aws-key-rotation.git
$ cd aws-key-rotation
~~~
![alt text](https://i.ibb.co/9b76FTv/2.png)

##### 2. Update the Ansible variable file - key.vars
~~~
access_key: "<>"
secret_key: "<>"
region: "<>"  #----> Example: "ap-south-1"
old_key: "<>"   #----> Upload this Pem file in the same directory with 400 Permission.
new_key: "<>"
system_user: "<>"
ssh_port: 22
~~~

access_key: Add your AWS access key

secret_key: Add your AWS secret key

region: Provide the AWS region in which EC2 instances are hosted. Example, "ap-south-1"

old_key: Add your SSH key name and upload the SSH private key in the same location with 400 permission

new_key: Add your new key name to be created

system_user: Add the System user

ssh_port: Add the SSH port number

![alt text](https://i.ibb.co/cwjyg2b/3.png)

##### 3. Run the Ansible playbook
~~~
$ ansible-playbook main.yml
~~~

![alt text](https://i.ibb.co/Vmb7rXZ/4.png)

##### 4. Try to SSH to Server using Old SSH key and New SSH key
~~~
$ ssh -i old-key.pem user@{server-IP}}
~~~
~~~
$ ssh -i new-key.pem ubuntu@{server-IP}}
~~~
![alt text](https://i.ibb.co/rcnVd9P/5.png)

## Further Information

The old keypair name remains same in EC2-AWS console. You cannot rename/change an existing keypair name which appears in the console even if that keypair no longer exists.
If you need to replace the ssh key without changing the key name, please refer the another playbook from my [Github repository](https://github.com/sebinxavi/aws-key-rotation-without-changing-keyname.git)

## Demonstration Video

<a href="https://www.youtube.com/watch?v=ocfrFrFc5X4" target="_blank">
 <img src="https://i.ibb.co/tqb49S4/AWS-KEY-ROTATION.png" alt="Watch the video" width="800" height="450" border="10" />
</a>
                                                                                           
## Author
Created by [@sebinxavi](https://www.linkedin.com/in/sebinxavi/) - feel free to contact me and advise as necessary!

<a href="mailto:sebin.xavi1@gmail.com"><img src="https://img.shields.io/badge/-sebin.xavi1@gmail.com-D14836?style=flat&logo=Gmail&logoColor=white"/></a>
<a href="https://www.linkedin.com/in/sebinxavi"><img src="https://img.shields.io/badge/-Linkedin-blue"/></a>
