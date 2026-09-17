# Project 13: Ansible Dynamic Assignments (Include) & Community Roles

This project demonstrates the implementation of **dynamic assignments** in Ansible using the `include_vars` module, alongside organizing environment-specific variables and integrating community roles from Ansible Galaxy into an automated enterprise deployment pipeline.

---

## Technical Overview

* **Repository:** `amarsaleem333/Project13-Dynamic-Assignments`
* **Configuration Management:** Ansible (Dynamic Assignments & Static Playbooks)
* **Target OS:** Ubuntu 24.04 LTS / Red Hat Enterprise Linux 8.10
* **Cloud Infrastructure:** AWS EC2 Instances (us-east-1)

---

## AWS Infrastructure Topology

The project infrastructure consists of dedicated AWS EC2 instances supporting Jenkins/Ansible orchestration and multi-node UAT server environments:

| Server / Host Role | Instance ID | Private IP | OS / Distribution | State |
| :--- | :--- | :--- | :--- | :--- |
| **Jenkins-Ansible Controller** | `i-0896842635656b392` | `172.31.32.54` | Ubuntu 24.04 LTS 
| **Web1-UAT** | `i-0609fd26cb83f372f` | `172.31.8.200` | RHEL 8.10 
| **Web2-UAT** | `i-0cfd71d050e9465e8` | `172.31.0.22` | RHEL 8.10 
---

## Repository Structure & File Layout

The repository is structured to separate static playbooks, dynamic assignments, environment variables, inventory files, and modular roles:

```text
Project13-Dynamic-Assignments/
├── dynamic-assignments/
│   └── env-vars.yml          # Dynamic inclusion logic using include_vars
├── env-vars/
│   ├── dev.yml               # Dev environment configuration variables
│   ├── stage.yml             # Stage environment configuration variables
│   ├── uat.yml               # UAT environment configuration variables (LB flags)
│   └── prod.yml              # Production environment configuration variables
├── inventory/
│   ├── dev.yml               # Inventory configuration for Dev
│   ├── stage.yml             # Inventory configuration for Stage
│   ├── uat.yml               # Inventory configuration for UAT nodes
│   └── prod.yml              # Inventory configuration for Production
├── playbooks/
│   └── site.yml              # Main entry point playbook referencing dynamic/static imports
├── roles/
│   ├── mysql/                # Community MySQL role (geerlingguy.mysql)
│   └── webserver/            # Webserver configuration roles
├── static-assignments/
│   ├── common.yml            # Base system configurations
│   ├── common-del.yml        # Utility teardown playbooks
│   ├── db.yml                # Database provisioning
│   ├── loadbalancers.yml     # Conditional Nginx/Apache LB playbook
│   └── uat-webservers.yml    # UAT Webserver task assignments
└── ansible.cfg               # Core Ansible behavior configuration

___________________________________________

Key Implementation Highlights

1. Dynamic Variable Inclusion (env-vars.yml)
Instead of static imports pre-processed at parse time, dynamic inclusions process statements during runtime. The env-vars.yml task dynamically loads environment files based on inventory conditions:

YAML
---
- name: Collate variables from env specific file
  hosts: all
  tasks:
    - name: Looping through list of available files
      include_vars: "{{ item }}"
      with_first_found:
        - files:
            - "{{ inventory_file | basename }}.yml"
            - "{{ inventory_file | basename }}"
            - "dev.yml"
          paths:
            - "{{ playbook_dir }}/../env-vars"
      tags:
        - always

2. Main Orchestration (playbooks/site.yml)
The main entry point ties dynamic variable inclusions with conditional load balancer configurations and webserver task execution:

YAML
---
- hosts: all
  name: Include dynamic variables 
  tasks:
    - import_playbook: ../static-assignments/common.yml 
    - include: ../dynamic-assignments/env-vars.yml
      tags:
        - always

- hosts: lb
  name: Loadbalancers assignment
  tasks:
    - import_playbook: ../static-assignments/loadbalancers.yml
      when: load_balancer_is_required

3. Conditional Load Balancer Switch
Environment-specific files (env-vars/uat.yml) dynamically toggle between Nginx and Apache deployment without altering core playbooks:

YAML
enable_nginx_lb: true
enable_apache_lb: false
load_balancer_is_required: true

Verification & Workspace Artifacts

VS Code Project Workspace: Full local directory structure and syntax validation matching GitHub remote state.

GitHub Synchronization: All feature branches merged to master with tracking across environments.

---


____________________________________

Jenkins-Ansible EC2 server

1. Server Environment Setup & Git Synchronization
Initialize the Git repository on the control server and create the feature branch

# Verify Git installation and navigate to workspace
git --version
cd ~/ansible-config-mgt

# Initialize and pull current project files
git init
git remote add origin https://github.com/amarsaleem333/Project13-Dynamic-Assignments.git
git pull origin master

# Create and switch to working feature branch
git checkout -b dynamic-assignments

Console Output:

ubuntu@ip-172.31-32-54:~/ansible-config-mgt$ git checkout -b dynamic-assignments
Switched to a new branch 'dynamic-assignments'

2. Folder Hierarchy & Dynamic File Construction
Build the dynamic directory layout and environment variable files

# Create dynamic assignments and environment directories
mkdir dynamic-assignments env-vars

# Create dynamic variable files
touch dynamic-assignments/env-vars.yml
touch env-vars/dev.yml env-vars/stage.yml env-vars/uat.yml env-vars/prod.yml

Console Output:

ubuntu@ip-172.31-32-54:~/ansible-config-mgt$ tree -L 2
.
├── dynamic-assignments
│   └── env-vars.yml
├── env-vars
│   ├── dev.yml
│   ├── prod.yml
│   ├── stage.yml
│   └── uat.yml
├── inventory
│   ├── dev.yml
│   ├── prod.yml
│   ├── stage.yml
│   └── uat.yml
├── playbooks
│   └── site.yml
└── static-assignments
    └── common.yml

3. Community Role Download via Ansible Galaxy
Download and install the community MySQL role directly into the project roles directory

# Navigate to roles directory and download MySQL role from Galaxy
cd roles
ansible-galaxy install geerlingguy.mysql

# Rename community role directory for structure standardization
mv geerlingguy.mysql mysql

ubuntu@ip-172.31-32-54:~/ansible-config-mgt/roles$ ansible-galaxy install geerlingguy.mysql
- downloading role 'mysql', via github website
- extracting geerlingguy.mysql to /home/ubuntu/ansible-config-mgt/roles/geerlingguy.mysql
- geerlingguy.mysql (3.4.2) was installed successfully
ubuntu@ip-172.31-32-54:~/ansible-config-mgt/roles$ mv geerlingguy.mysql mysql

4. Running Playbook Execution & Inventory VerificationExecute the master site.yml playbook against target UAT hosts (Web1-UAT and Web2-UAT):

# Test connection and run site.yml with UAT inventory
ansible-playbook -i inventory/uat.yml playbooks/site.yml

Console Output:

PLAY [Include dynamic variables] *********************************************************************************

TASK [Gathering Facts] *******************************************************************************************
ok: [172.31.8.200]
ok: [172.31.0.22]

TASK [Looping through list of available files] ******************************************************************
included: /home/ubuntu/ansible-config-mgt/env-vars/uat.yml for 172.31.8.200, 172.31.0.22

PLAY [Loadbalancers assignment] **********************************************************************************

TASK [Nginx : Install Nginx] *************************************************************************************
changed: [172.31.8.200]
changed: [172.31.0.22]

PLAY RECAP *******************************************************************************************************
172.31.0.22                : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
172.31.8.200               : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

5. Git Commit & Remote Synchronization

# Stage, commit, and push changes to GitHub
git add .
git commit -m "Add dynamic-assignments, env-vars, and community mysql role"
git push origin dynamic-assignments

Console Output:

ubuntu@ip-172.31-32-54:~/ansible-config-mgt$ git push origin dynamic-assignments
Enumerating objects: 21, done.
Counting objects: 100% (21/21), done.
Writing objects: 100% (21/21), 4.2 KiB | 4.20 MiB/s, done.
Total 21 (delta 5), reused 0 (delta 0)
To https://github.com/amarsaleem333/Project13-Dynamic-Assignments.git
 * [new branch]      dynamic-assignments -> dynamic-assignments




