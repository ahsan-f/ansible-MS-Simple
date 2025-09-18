# ansible-MS-Simple
Setting up an Ansible lab with Docker, Python, and SSH, orchestrated by GitHub Actions for a CI/CD pipeline, is an excellent way to learn automation in an isolated, repeatable environment. Here's a lab plan to achieve this.

Lab Overview

The goal is to create a CI/CD pipeline that automatically provisions a simple environment every time you push changes to your GitHub repository. The environment will consist of:
A controller container with Ansible and Python installed.
Two target containers that the controller can manage via SSH.
All of this will be defined in code (Infrastructure as Code) using a Dockerfile and docker-compose.yml, with the execution automated by GitHub Actions.

1. Repository Setup

Create a new GitHub repository to hold your lab code. The file structure should look like this:



ansible-docker-lab/
├── .github/
│   └── workflows/
│       └── ci-cd.yml
├── docker-compose.yml
├── Dockerfile.controller
├── Dockerfile.target
├── ansible/
│   ├── inventory.ini
│   └── playbooks/
│       └── site.yml
└── requirements.txt



2a.Requirement.txt
ansible==9.1.0
2b. Docker Files

You will need two Dockerfiles to build your custom images: one for the controller and one for the target nodes.

Dockerfile.controller

This file builds the Ansible controller image.

Dockerfile
# Use a lightweight Linux base image
FROM ubuntu:22.04

# Install required packages
RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    openssh-client \
    git \
    --no-install-recommends \
    && rm -rf /var/lib/apt/lists/*

# Install Ansible via pip
COPY ./ansible/requirements.txt .
RUN pip3 install -r requirements.txt

# Set up SSH directory for keys
RUN mkdir -p /root/.ssh
RUN chmod 700 /root/.ssh

# Expose a volume for SSH keys
VOLUME /root/.ssh

# Set working directory
WORKDIR /ansible

# This command keeps the container running indefinitely
CMD ["tail", "-f", "/dev/null"]



Dockerfile.target

This file builds the target node image. It must have an SSH server running.

FROM ubuntu:22.04

# Install required packages
# The flags must be on the same line as the apt-get install command
RUN apt-get update && apt-get install -y \
    openssh-server \
    python3 \
    sudo \
    --no-install-recommends \
    && rm -rf /var/lib/apt/lists/*

# Set up SSH
RUN mkdir /var/run/sshd
EXPOSE 22

# Create a non-root user for Ansible to connect as
RUN useradd -ms /bin/bash ansible
RUN echo "ansible:password" | chpasswd
RUN usermod -aG sudo ansible
RUN echo "ansible ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

# Copy the SSH public key from a volume (to be mounted by docker-compose)
COPY authorized_keys /home/ansible/.ssh/authorized_keys
RUN chown -R ansible:ansible /home/ansible/.ssh
RUN chmod 700 /home/ansible/.ssh
RUN chmod 600 /home/ansible/.ssh/authorized_keys

# Run SSH service
CMD ["/usr/sbin/sshd", "-D"]





3. Docker Compose

The docker-compose.yml file defines the entire lab environment.
version: '3.8'

services:
  controller:
    build:
      context: .
      dockerfile: Dockerfile.controller
    container_name: ansible-controller
    volumes:
      - ./ansible:/ansible
      - ssh_key_volume:/root/.ssh
    networks:
      ansible_network:
        ipv4_address: 172.18.0.2

  target1:
    build:
      context: .
      dockerfile: Dockerfile.target
    container_name: target-node-1
    volumes:
      - ssh_key_volume:/home/ansible/.ssh
    networks:
      ansible_network:
        ipv4_address: 172.18.0.3

  target2:
    build:
      context: .
      dockerfile: Dockerfile.target
    container_name: target-node-2
    volumes:
      - ssh_key_volume:/home/ansible/.ssh
    networks:
      ansible_network:
        ipv4_address: 172.18.0.4

networks:
  ansible_network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.18.0.0/16

volumes:
  ssh_key_volume:



4. Ansible Components


ansible/inventory.ini
[targets]
target-node-1 ansible_host=172.18.0.3
target-node-2 ansible_host=172.18.0.4

[targets:vars]
ansible_user=ansible
ansible_ssh_private_key_file=/root/.ssh/id_rsa
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_common_args='-o StrictHostKeyChecking=no'



ansible/playbooks/site.yml

A good example is a playbook that installs and starts a web server (like Apache on Ubuntu or Nginx on CentOS). This demonstrates a more complex and practical use case for Ansible.

YAML
---
- name: Deploy a basic web server
  hosts: targets
  become: yes
  tasks:
    - name: Install Apache on Debian/Ubuntu
      ansible.builtin.apt:
        name: apache2
        state: present
      when: ansible_os_family == "Debian"

    - name: Start and enable Apache service
      ansible.builtin.service:
        name: apache2
        state: started
        enabled: yes
      when: ansible_os_family == "Debian"

    - name: Install Nginx on CentOS/Red Hat
      ansible.builtin.dnf:
        name: nginx
        state: present
      when: ansible_os_family == "RedHat"

    - name: Start and enable Nginx service
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: yes
      when: ansible_os_family == "RedHat"
How It Works
become: yes: This tells Ansible to use sudo to run the tasks with root privileges, which is necessary for installing and managing services.
Conditional Logic: The when statements check the ansible_os_family variable, a fact automatically gathered by Ansible. This allows the playbook to run on different Linux distributions without modification.
If the target node is Debian (or Ubuntu), it will use the ansible.builtin.apt module to install apache2 and the ansible.builtin.service module to ensure it's running.
If the target node is Red Hat (or CentOS), it will use the ansible.builtin.dnf module to install nginx and the ansible.builtin.service module to start it.
This new playbook is a great example for your lab as it demonstrates how to handle multiple operating systems and perform a common, real-world DevOps task.



5. GitHub Actions Workflow

This workflow will spin up the lab, run the playbook, and then tear down the environment.

.github/workflows/ci-cd.yml


YAML
name: Ansible Docker Lab CI/CD

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  run-ansible-lab:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      - name: Install Docker Compose
        run: |
             sudo apt-get update
             sudo apt-get install -y docker-compose
      - name: Generate SSH key pair
        run: |
          mkdir -p ./.ssh
          ssh-keygen -t rsa -N "" -f ./.ssh/id_rsa
          mv ./.ssh/id_rsa.pub ./authorized_keys
          
      - name: Log SSH keys
        run: |
          echo "Private Key:"
          cat ./.ssh/id_rsa
          echo "Public Key:"
          cat ./authorized_keys
          
      - name: Build and start Docker containers
        run: |
          docker-compose -f docker-compose.yml up --build -d
          
      - name: Wait for containers to be ready
        run: sleep 10

      - name: Copy SSH key to controller container
        run: docker cp ./.ssh/id_rsa ansible-controller:/root/.ssh/id_rsa
        
      - name: Run Ansible playbook
        run: |
          docker exec ansible-controller ansible-playbook -i /ansible/inventory.ini /ansible/playbooks/site.yml
          
      - name: Teardown lab
        if: always()
        run: docker-compose -f docker-compose.yml down



Explanation of the CI/CD Pipeline

Trigger: The workflow is triggered on a push to the main branch or manually via workflow_dispatch.
Generate Keys: A new SSH key pair is generated on the GitHub Actions runner. This is a crucial step as it establishes trust between the controller and the target nodes.
Build and Start: docker-compose up --build -d reads the docker-compose.yml file, builds the images from your Dockerfiles, and starts the three containers on an isolated network. The ssh_key_volume ensures the public key is shared with the target nodes.
Copy Key: The private key is copied from the runner into the ansible-controller container, allowing it to authenticate with the target nodes.
Run Playbook: The docker exec command enters the ansible-controller container and runs the ansible-playbook command, executing the automation.
Teardown: The if: always() condition ensures the docker-compose down command runs regardless of whether the playbook succeeded or failed, cleaning up the environment.

