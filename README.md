# 🚀 GCP DevOps Infrastructure — Ansible Automation Runbook

> **Production-style DevOps lab documentation for Ubuntu 24.04 LTS on Google Cloud Platform**
>
> This runbook provisions the complete lab using **Ansible**, with the Jenkins server acting as the Ansible controller.

---

## 📌 1. Lab Architecture

| Server | IP Address | Host Group | Primary Role |
|---|---:|---|---|
| Jenkins | `Jenkins Ex-IP` | `master` | Jenkins + Ansible Controller |
| Docker | `Docker Ex-IP` | `docker` | Docker workloads |
| Monitoring | `Monitoring Ex-IP` | `monitoring` | Prometheus + Grafana |
| SonarQube | `Sonar Ex-IP` | `sonar` | SonarQube + PostgreSQL |

### Architecture

```text
                         ┌──────────────────────────────┐
                         │        GCP VPC Network       │
                         └──────────────┬───────────────┘
                                        │
                     ┌──────────────────▼──────────────────┐
                     │       Jenkins / Ansible Controller  │
                     │       136.115.122.174               │
                     │       Ubuntu 24.04 LTS              │
                     │       Jenkins + Java + Maven        │
                     └───────────────┬─────────────────────┘
                                     │ SSH / Ansible
                ┌────────────────────┼────────────────────┐
                │                    │                    │
        ┌───────▼────────┐   ┌───────▼────────┐   ┌───────▼────────┐
        │ Docker Server  │   │ Monitoring      │   │ SonarQube      │
        │ 34.44.81.210   │   │ 136.112.98.192  │   │ 34.27.72.86    │
        │ Docker          │   │ Prometheus      │   │ SonarQube      │
        │                 │   │ Grafana         │   │ PostgreSQL     │
        └─────────────────┘   └─────────────────┘   └────────────────┘
```

---

# 🔐 2. SSH Passwordless Authentication — Without Interactive Fingerprint Prompts

Ansible runs from the **Jenkins server (`136.115.122.174`)**, which acts as the Ansible controller. This section configures SSH public-key authentication to the Docker, Monitoring, and SonarQube servers and avoids the interactive `Are you sure you want to continue connecting (yes/no/[fingerprint])?` prompt.

**Important security note:** SSH host-key checking should not be disabled blindly. The configuration below uses `StrictHostKeyChecking accept-new`: SSH automatically saves a host key the first time it sees it, but still rejects a changed key later. For production, verify each host-key fingerprint through a trusted channel before accepting it.

## 2.1 Confirm the SSH usernames and server addresses

Use the IP addresses and usernames configured for this lab:

| Destination | IP address | SSH username |
|---|---|---|
| Docker | `34.44.81.210` | `docker` |
| Monitoring | `136.112.98.192` | `monitoring` |
| SonarQube | `34.27.72.86` | `sonar` |
| Jenkins/controller (only if you intentionally need SSH to itself) | `136.115.122.174` | `master` |

Run the following commands on the **Jenkins/Ansible controller**, logged in as the same Linux account that will run `ansible` and `ansible-playbook`. Do not generate the key as one user and run Ansible as another user (for example, do not mix your login account with the `jenkins` service account unless you deliberately configure the key for that account).

## 2.2 Install the SSH client utilities

```bash
sudo apt update
sudo apt install -y openssh-client
```

## 2.3 Create an SSH key pair

Check whether a key already exists:

```bash
ls -l ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub
```

If both files already exist and you intend to reuse that key, do not overwrite them. Otherwise, create a key:

```bash
ssh-keygen -t ed25519 -C "jenkins-ansible-controller"
```

Press **Enter** to accept the default path (`~/.ssh/id_ed25519`). For unattended lab automation, you can leave the passphrase empty when prompted; a passphrase-protected key is safer but requires an agent or another secure way to unlock it.

Set permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

## 2.4 Configure SSH to accept a new host key without asking yes/no

Create or edit the SSH client configuration on the controller:

```bash
nano ~/.ssh/config
```

Add the following. If the file already contains host entries, append these entries rather than deleting existing configuration:

```sshconfig
Host 34.44.81.210
    User docker
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new

Host 136.112.98.192
    User monitoring
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new

Host 34.27.72.86
    User sonar
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new
```

Save with `Ctrl+O`, press Enter, then exit with `Ctrl+X`. Set the configuration permissions:

```bash
chmod 600 ~/.ssh/config
```

This setting prevents the first-connection yes/no prompt by automatically adding a previously unseen host key to `~/.ssh/known_hosts`. It does **not** bypass a changed-host-key warning. If a server is rebuilt and its host key legitimately changes, verify the new fingerprint before removing the old entry.

## 2.5 Copy the public key to each destination server

Run these commands on the controller. You may be prompted for the destination user's password during this initial setup; after the key is installed, SSH should use the key instead.

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub docker@34.44.81.210
ssh-copy-id -i ~/.ssh/id_ed25519.pub monitoring@136.112.98.192
ssh-copy-id -i ~/.ssh/id_ed25519.pub sonar@34.27.72.86
```

If the initial connection prints a fingerprint prompt, the `accept-new` configuration above should suppress the yes/no question. Before the first connection to an important or production server, independently verify the server fingerprint using a trusted channel.

**Do not copy the key to `master@136.115.122.174` unless you specifically need the controller to SSH to itself.** Ansible does not need a self-SSH connection just to manage the other three servers.

If `ssh-copy-id` is unavailable, install it with `sudo apt install -y openssh-client`. If key installation still fails, confirm that the destination username exists and that SSH password authentication or another approved initial access method is available.

## 2.6 Test passwordless SSH and fingerprint-prompt behavior

Run on the controller:

```bash
ssh docker@34.44.81.210
ssh monitoring@136.112.98.192
ssh sonar@34.27.72.86
```

Each connection should log in without asking for the account password once the public key is installed. Exit each remote session with:

```bash
exit
```

Check that the host keys are recorded:

```bash
ssh-keygen -F 34.44.81.210
ssh-keygen -F 136.112.98.192
ssh-keygen -F 34.27.72.86
```

## 2.7 Optional: remove an obsolete host-key entry

Only do this when a server has been rebuilt or its host key has changed and you have verified the new fingerprint through a trusted channel. Do not remove entries just to silence a warning.

```bash
ssh-keygen -R 34.44.81.210
ssh-keygen -R 136.112.98.192
ssh-keygen -R 34.27.72.86
```

Then reconnect using the `accept-new` configuration so the new key is recorded.

## 2.8 Ensure Ansible uses the same SSH settings

In the `hosts` inventory file, use the actual server addresses and usernames. Add `ansible_ssh_common_args` so Ansible's SSH connections also avoid an interactive first-connection prompt:

```ini
[docker]
34.44.81.210 ansible_user=docker

[monitoring]
136.112.98.192 ansible_user=monitoring

[sonar]
34.27.72.86 ansible_user=sonar

[all:vars]
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_ssh_common_args='-o StrictHostKeyChecking=accept-new'
ansible_python_interpreter=/usr/bin/python3
```

The Jenkins/controller host is intentionally omitted from the managed inventory because it is the controller itself. If you intentionally manage it as a remote host, add a separate `[master]` entry only after confirming the correct Linux username and key setup.

## 2.9 Test Ansible connectivity

From the controller, run:

```bash
ansible all -i hosts -m ping
```

Expected result for each managed host:

```text
SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

If Ansible asks for a password or fails, test SSH directly first. Then inspect permissions and the verbose connection log:

```bash
ls -la ~/.ssh
ssh -vvv docker@34.44.81.210
ansible all -i hosts -m ping -vvv
```

Remember: SSH key-based login removes the remote account-password prompt; `accept-new` removes the interactive first-time yes/no prompt. These are separate settings. Ansible tasks that use `become: yes` may still ask for a sudo password unless sudo permissions are configured appropriately.

---

# 🔑 4. Sudo Permissions

If the Ansible user must execute administrative commands without entering a password, configure sudo carefully.

Edit sudoers:

```bash
sudo visudo
```

Example:

```text
docker ALL=(ALL) NOPASSWD: ALL
monitoring ALL=(ALL) NOPASSWD: ALL
sonar ALL=(ALL) NOPASSWD: ALL
master ALL=(ALL) NOPASSWD: ALL
```

> ⚠️ `NOPASSWD: ALL` provides full administrative access. Use it only in a controlled lab or where your security policy permits it.


# 📁 5. Ansible Installation

Run these commands on the Jenkins/Ansible controller.

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install ansible -y
ansible --version
```

Create the working directory:

```bash
sudo mkdir -p ~/ansible
cd ~/ansible
```

---

# 🧾 6. Hosts Inventory

Create the inventory file named **`hosts`**:

```bash
sudo nano hosts
```

Use:

```ini
[docker]
34.44.81.210 ansible_user=docker

[monitoring]
136.112.98.192 ansible_user=monitoring

[sonar]
34.27.72.86 ansible_user=sonar

[all:vars]
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_ssh_common_args='-o StrictHostKeyChecking=accept-new'
ansible_python_interpreter=/usr/bin/python3
```

---

# 🧪 7. Test Ansible Connectivity

Test all servers:

```bash
ansible all -i hosts -m ping
```

Expected:

```text
SUCCESS
"ping": "pong"
```

Test individual groups:

```bash
ansible docker -i hosts -m ping
ansible monitoring -i hosts -m ping
ansible sonar -i hosts -m ping
ansible master -i hosts -m ping
```

---

# 📦 8. Project Structure

Recommended structure:

```text
~/ansible/
│
├── hosts
├── Basic-Packages-install.yml
├── Docker-install.yml
├── Monitoring-install.yml
├── SonarQube-install.yml
└── Jenkins-install.yml
```

Move into the project directory:

```bash
cd ~/ansible
```

Check the files:

```bash
ls -lh
```

---

# 🧰 9. Basic Packages Installation

Create the playbook:

```bash
sudo nano Basic-Packages-install.yml
```

Use:

```yaml
---
- name: Common Configuration
  hosts: docker,monitoring,sonar,master
  become: yes

  tasks:
    - name: Update Apt Cache
      ansible.builtin.apt:
        update_cache: yes

    - name: Upgrade Packages
      ansible.builtin.apt:
        upgrade: dist

    - name: Install Common Packages
      ansible.builtin.apt:
        name:
          - git
          - curl
          - wget
          - unzip
          - vim
          - tree
          - zip
          - net-tools
          - htop
          - software-properties-common
        state: present

    - name: Display Hostname
      ansible.builtin.command: hostname
      register: hostname_output
      changed_when: false

    - name: Show Hostname
      ansible.builtin.debug:
        var: hostname_output.stdout
```

Validate:

```bash
ansible-playbook -i hosts Basic-Packages-install.yml --syntax-check
```

Run:

```bash
ansible-playbook -i hosts Basic-Packages-install.yml
```

---

# 🐳 10. Docker Server Installation

Create:

```bash
sudo nano Docker-install.yml
```

Use:

```yaml
---
- name: Docker Server Configuration
  hosts: docker
  become: yes

  tasks:
    - name: Install Docker
      ansible.builtin.apt:
        name: docker.io
        state: present
        update_cache: yes

    - name: Enable Docker
      ansible.builtin.service:
        name: docker
        enabled: yes

    - name: Start Docker
      ansible.builtin.service:
        name: docker
        state: started

    - name: Add docker user to docker group
      ansible.builtin.user:
        name: docker
        groups: docker
        append: yes

    - name: Verify Docker Version
      ansible.builtin.command: docker --version
      register: docker_version
      changed_when: false

    - name: Display Docker Version
      ansible.builtin.debug:
        var: docker_version.stdout
```

Validate:

```bash
ansible-playbook -i hosts Docker-install.yml --syntax-check
```

Run:

```bash
ansible-playbook -i hosts Docker-install.yml
```

Verify:

```bash
ansible docker -i hosts -m command -a "docker --version"
```

```bash
ansible docker -i hosts -m command -a "docker ps"
```

> **Important:** After adding a user to the `docker` group, a new login session may be required for the group membership to take effect.

---

# 📊 11. Monitoring Server — Prometheus + Grafana

Create:

```bash
sudo nano Monitoring-install.yml
```

Use:

```yaml
---
- name: Monitoring Server Configuration
  hosts: monitoring
  become: yes

  tasks:
    - name: Install Docker
      ansible.builtin.apt:
        name: docker.io
        state: present
        update_cache: yes

    - name: Enable Docker
      ansible.builtin.service:
        name: docker
        enabled: yes

    - name: Start Docker
      ansible.builtin.service:
        name: docker
        state: started

    - name: Add monitoring user to docker group
      ansible.builtin.user:
        name: monitoring
        groups: docker
        append: yes

    - name: Remove Existing Prometheus
      ansible.builtin.command: docker rm -f prometheus
      failed_when: false
      changed_when: false

    - name: Remove Existing Grafana
      ansible.builtin.command: docker rm -f grafana
      failed_when: false
      changed_when: false

    - name: Pull Prometheus Image
      ansible.builtin.command: docker pull prom/prometheus:latest

    - name: Run Prometheus
      ansible.builtin.command:
        argv:
          - docker
          - run
          - -d
          - --name
          - prometheus
          - --restart
          - always
          - -p
          - 9090:9090
          - prom/prometheus:latest

    - name: Pull Grafana Image
      ansible.builtin.command: docker pull grafana/grafana:latest

    - name: Run Grafana
      ansible.builtin.command:
        argv:
          - docker
          - run
          - -d
          - --name
          - grafana
          - --restart
          - always
          - -p
          - 3000:3000
          - grafana/grafana:latest

    - name: Display Running Containers
      ansible.builtin.command: docker ps
      register: output
      changed_when: false

    - name: Show Running Containers
      ansible.builtin.debug:
        var: output.stdout_lines
```

Validate:

```bash
ansible-playbook -i hosts Monitoring-install.yml --syntax-check
```

Run:

```bash
ansible-playbook -i hosts Monitoring-install.yml
```

Verify:

```bash
ansible monitoring -i hosts -b -m command -a "docker ps"
```

Prometheus:

```text
http://Monitoring External IP:9090
```

Grafana:

```text
http://Monitoring External IP:3000
```

---

# 🔍 12. SonarQube + PostgreSQL

> SonarQube requires Java and PostgreSQL. The following playbook installs Java 21, PostgreSQL, the SonarQube application, systemd configuration, and required Linux kernel settings.

Create:

```bash
sudo nano SonarQube-install.yml
```

Use:

```yaml
---
- name: Install and Configure SonarQube Server
  hosts: sonar
  become: yes
  gather_facts: yes

  vars:
    sonar_version: "25.7.0.110598"
    sonar_download_url: "https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-25.7.0.110598.zip"

    sonar_home: /opt/sonarqube
    sonar_user: sonar
    sonar_group: sonar

    postgres_db: sonarqube
    postgres_user: sonar
    postgres_password: sonar123

  tasks:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install Java 21
      ansible.builtin.apt:
        name: openjdk-21-jdk
        state: present

    - name: Install PostgreSQL
      ansible.builtin.apt:
        name:
          - postgresql
          - postgresql-contrib
        state: present

    - name: Enable PostgreSQL
      ansible.builtin.service:
        name: postgresql
        state: started
        enabled: yes

    - name: Create Sonar PostgreSQL User
      ansible.builtin.shell: |
        sudo -u postgres psql -tAc "SELECT 1 FROM pg_roles WHERE rolname='{{ postgres_user }}'" | grep -q 1 || \
        sudo -u postgres psql -c "CREATE USER {{ postgres_user }} WITH PASSWORD '{{ postgres_password }}';"
      args:
        executable: /bin/bash

    - name: Create Sonar Database
      ansible.builtin.shell: |
        sudo -u postgres psql -lqt | cut -d \| -f1 | grep -qw {{ postgres_db }} || \
        sudo -u postgres createdb -O {{ postgres_user }} {{ postgres_db }}
      args:
        executable: /bin/bash

    - name: Install unzip and wget
      ansible.builtin.apt:
        name:
          - unzip
          - wget
        state: present

    - name: Download SonarQube
      ansible.builtin.get_url:
        url: "{{ sonar_download_url }}"
        dest: /tmp/sonarqube.zip
        mode: "0644"

    - name: Extract SonarQube
      ansible.builtin.unarchive:
        src: /tmp/sonarqube.zip
        dest: /opt
        remote_src: yes
        creates: "/opt/sonarqube-{{ sonar_version }}"

    - name: Rename SonarQube Folder
      ansible.builtin.command:
        cmd: "mv /opt/sonarqube-{{ sonar_version }} {{ sonar_home }}"
        creates: "{{ sonar_home }}"

    - name: Create Sonar Group
      ansible.builtin.group:
        name: "{{ sonar_group }}"
        state: present

    - name: Create Sonar User
      ansible.builtin.user:
        name: "{{ sonar_user }}"
        group: "{{ sonar_group }}"
        shell: /bin/bash
        system: yes
        create_home: no

    - name: Change Ownership
      ansible.builtin.file:
        path: "{{ sonar_home }}"
        owner: "{{ sonar_user }}"
        group: "{{ sonar_group }}"
        recurse: yes

    - name: Configure sonar.properties
      ansible.builtin.blockinfile:
        path: "{{ sonar_home }}/conf/sonar.properties"
        marker: "# {mark} ANSIBLE MANAGED BLOCK"
        block: |
          sonar.jdbc.username={{ postgres_user }}
          sonar.jdbc.password={{ postgres_password }}
          sonar.jdbc.url=jdbc:postgresql://localhost/{{ postgres_db }}

          sonar.web.host=0.0.0.0
          sonar.web.port=9000

    - name: Configure sonar.sh
      ansible.builtin.lineinfile:
        path: "{{ sonar_home }}/bin/linux-x86-64/sonar.sh"
        regexp: "^RUN_AS_USER="
        line: "RUN_AS_USER={{ sonar_user }}"

    - name: Configure vm.max_map_count
      ansible.posix.sysctl:
        name: vm.max_map_count
        value: "524288"
        state: present
        reload: yes

    - name: Configure fs.file-max
      ansible.posix.sysctl:
        name: fs.file-max
        value: "131072"
        state: present
        reload: yes

    - name: Create SonarQube Service
      ansible.builtin.copy:
        dest: /etc/systemd/system/sonarqube.service
        mode: "0644"
        content: |
          [Unit]
          Description=SonarQube
          After=network.target postgresql.service

          [Service]
          Type=forking
          User=sonar
          Group=sonar
          ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
          ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
          Restart=always
          LimitNOFILE=65536
          LimitNPROC=4096

          [Install]
          WantedBy=multi-user.target

    - name: Reload systemd
      ansible.builtin.systemd:
        daemon_reload: yes

    - name: Enable SonarQube
      ansible.builtin.systemd:
        name: sonarqube
        enabled: yes

    - name: Start SonarQube
      ansible.builtin.systemd:
        name: sonarqube
        state: started

    - name: Wait for Port 9000
      ansible.builtin.wait_for:
        port: 9000
        delay: 20
        timeout: 300

    - name: Verify Java
      ansible.builtin.command: java -version
      register: java_version
      changed_when: false

    - name: Verify PostgreSQL
      ansible.builtin.command: psql --version
      register: postgres_version
      changed_when: false

    - name: Verify SonarQube
      ansible.builtin.command: systemctl is-active sonarqube
      register: sonar_status
      changed_when: false

    - name: Verify Port
      ansible.builtin.shell: ss -tulpn | grep 9000
      register: sonar_port
      changed_when: false

    - name: Display SonarQube Status
      ansible.builtin.debug:
        msg:
          - "Java: {{ java_version.stderr_lines }}"
          - "PostgreSQL: {{ postgres_version.stdout }}"
          - "SonarQube: {{ sonar_status.stdout }}"
          - "Port 9000: {{ sonar_port.stdout_lines }}"

    - name: Success Message
      ansible.builtin.debug:
        msg:
          - "======================================"
          - " SonarQube Installation Successful "
          - " URL: http://{{ inventory_hostname }}:9000"
          - " Default username: admin"
          - " Default password: admin"
          - "======================================"
```

Validate:

```bash
ansible-playbook -i hosts SonarQube-install.yml --syntax-check
```

Run:

```bash
ansible-playbook -i hosts SonarQube-install.yml
```

Verify:

```bash
ansible sonar -i hosts -b -m command -a "systemctl is-active sonarqube"
```

```bash
ansible sonar -i hosts -m command -a "java -version"
```

```bash
ansible sonar -i hosts -m command -a "psql --version"
```

SonarQube:

```text
http://34.27.72.86:9000
```

---

# ☕ 13. Jenkins Server

Jenkins is installed only on the **`master`** server.

The Jenkins server should contain:

- Java 21
- Git
- Maven
- Docker
- Jenkins
- Ansible

Create:

```bash
nano Jenkins-install.yml
```

Use:

```yaml
---
- name: Jenkins Server Configuration
  hosts: master
  become: yes

  tasks:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: yes

    - name: Install Java 21
      ansible.builtin.apt:
        name: openjdk-21-jdk
        state: present

    - name: Install Git
      ansible.builtin.apt:
        name: git
        state: present

    - name: Install Maven
      ansible.builtin.apt:
        name: maven
        state: present

    - name: Install Docker
      ansible.builtin.apt:
        name: docker.io
        state: present

    - name: Enable Docker
      ansible.builtin.service:
        name: docker
        enabled: yes

    - name: Start Docker
      ansible.builtin.service:
        name: docker
        state: started

    - name: Add Jenkins user to Docker group
      ansible.builtin.user:
        name: jenkins
        groups: docker
        append: yes

    - name: Install required Jenkins dependencies
      ansible.builtin.apt:
        name:
          - curl
          - gnupg
          - fontconfig
        state: present

    - name: Add Jenkins repository key
      ansible.builtin.get_url:
        url: https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
        dest: /etc/apt/keyrings/jenkins-keyring.asc
        mode: "0644"

    - name: Add Jenkins repository
      ansible.builtin.apt_repository:
        repo: "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/"
        filename: jenkins
        state: present

    - name: Update apt cache after Jenkins repository
      ansible.builtin.apt:
        update_cache: yes

    - name: Install Jenkins
      ansible.builtin.apt:
        name: jenkins
        state: present

    - name: Enable Jenkins
      ansible.builtin.systemd:
        name: jenkins
        enabled: yes

    - name: Start Jenkins
      ansible.builtin.systemd:
        name: jenkins
        state: started

    - name: Wait for Jenkins port
      ansible.builtin.wait_for:
        port: 8080
        delay: 10
        timeout: 180

    - name: Verify Java
      ansible.builtin.command: java -version
      register: java_version
      changed_when: false

    - name: Verify Git
      ansible.builtin.command: git --version
      register: git_version
      changed_when: false

    - name: Verify Maven
      ansible.builtin.command: mvn -version
      register: maven_version
      changed_when: false

    - name: Verify Docker
      ansible.builtin.command: docker --version
      register: docker_version
      changed_when: false

    - name: Verify Jenkins
      ansible.builtin.command: systemctl is-active jenkins
      register: jenkins_status
      changed_when: false

    - name: Display Jenkins Status
      ansible.builtin.debug:
        msg:
          - "Java: {{ java_version.stderr_lines }}"
          - "Git: {{ git_version.stdout }}"
          - "Maven: {{ maven_version.stdout_lines }}"
          - "Docker: {{ docker_version.stdout }}"
          - "Jenkins: {{ jenkins_status.stdout }}"
          - "Jenkins URL: http://{{ inventory_hostname }}:8080"
```

Validate:

```bash
ansible-playbook -i hosts Jenkins-install.yml --syntax-check
```

Run:

```bash
ansible-playbook -i hosts Jenkins-install.yml
```

Verify:

```bash
ansible master -i hosts -m command -a "java -version"
```

```bash
ansible master -i hosts -m command -a "git --version"
```

```bash
ansible master -i hosts -m command -a "mvn -version"
```

```bash
ansible master -i hosts -b -m command -a "docker --version"
```

```bash
ansible master -i hosts -b -m command -a "systemctl is-active jenkins"
```

Jenkins:

```text
http://136.115.122.174:8080
```

---

# 🔄 14. Recommended Execution Order

Run the playbooks in this order:

### Step 1 — Test connectivity

```bash
ansible all -i hosts -m ping
```

### Step 2 — Basic packages

```bash
ansible-playbook -i hosts Basic-Packages-install.yml
```

### Step 3 — Docker server

```bash
ansible-playbook -i hosts Docker-install.yml
```

### Step 4 — Monitoring

```bash
ansible-playbook -i hosts Monitoring-install.yml
```

### Step 5 — SonarQube

```bash
ansible-playbook -i hosts SonarQube-install.yml
```

### Step 6 — Jenkins

```bash
ansible-playbook -i hosts Jenkins-install.yml
```

---

# 🔎 15. Final Infrastructure Verification

### Check all hosts

```bash
ansible all -i hosts -m ping
```

### Check Docker server

```bash
ansible docker -i hosts -m command -a "docker --version"
```

### Check monitoring

```bash
ansible monitoring -i hosts -b -m command -a "docker ps"
```

### Check SonarQube

```bash
ansible sonar -i hosts -b -m command -a "systemctl is-active sonarqube"
```

### Check Jenkins

```bash
ansible master -i hosts -b -m command -a "systemctl is-active jenkins"
```

### Check Java

```bash
ansible master -i hosts -m shell -a "java -version"
ansible sonar -i hosts -m shell -a "java -version"
```

### Check Maven

```bash
ansible master -i hosts -m command -a "mvn -version"
```

---

# 🌐 16. Application URLs

| Application | URL |
|---|---|
| Jenkins | `http://Jenkins External IP:8080` |
| SonarQube | `http://Sonar External IP:9000` |
| Prometheus | `http://Prometheus External IP:9090` |
| Grafana | `http://Grafana External IP:3000` |

> Ensure the corresponding GCP VPC firewall rules allow TCP ports `8080`, `9000`, `9090`, and `3000` from the required source networks.

---

# 🔥 17. GCP Firewall Ports

Example GCP firewall rule:

```bash
gcloud compute firewall-rules create devops-lab-ports \
  --network=default \
  --allow=tcp:3000,tcp:8080,tcp:9000,tcp:9090 \
  --source-ranges=0.0.0.0/0
```

> ⚠️ `0.0.0.0/0` exposes the ports publicly. For a real environment, restrict `--source-ranges` to trusted IP ranges.

---

# 🛠️ 18. Common Troubleshooting

### Ansible SSH failure

```bash
ansible all -i hosts -m ping -vvv
```

Check:

```bash
ls -la ~/.ssh
```

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

### Docker permission denied

Check group:

```bash
id monitoring
```

or:

```bash
id jenkins
```

Check Docker socket:

```bash
ls -l /var/run/docker.sock
```

Test using sudo:

```bash
sudo docker ps
```

After group membership changes, start a new SSH session.

### Jenkins service status

```bash
sudo systemctl status jenkins
```

Logs:

```bash
sudo journalctl -u jenkins -n 100 --no-pager
```

### SonarQube service status

```bash
sudo systemctl status sonarqube
```

Logs:

```bash
sudo journalctl -u sonarqube -n 100 --no-pager
```

### Check port

```bash
sudo ss -lntp | grep -E '3000|8080|9000|9090'
```

---

# 🔐 19. Security Notes

Never commit private keys or secrets to GitHub.

Add these to `.gitignore`:

```gitignore
*.json
*.pem
*.key
id_ed25519
id_ed25519.pub
.env
terraform.tfstate
terraform.tfstate.*
```

For production environments:

- Use Google Cloud IAM and service accounts appropriately.
- Prefer short-lived credentials or Workload Identity where possible.
- Store passwords in Ansible Vault or a secret manager.
- Do not keep the PostgreSQL password directly in a public playbook.
- Restrict GCP firewall source ranges.
- Avoid exposing Jenkins, Grafana, Prometheus, and SonarQube directly to the internet.
- Use HTTPS and reverse proxies for production deployments.

---

# 🚀 20. Next CI/CD Stage

Once the infrastructure is working, the recommended pipeline is:

```text
Developer
    │
    ▼
GitHub
    │
    ▼
Jenkins
    │
    ├── Git Checkout
    │
    ├── Maven Build
    │
    ├── Unit Tests
    │
    ├── SonarQube Code Analysis
    │
    ├── Docker Build
    │
    ├── Security Scan
    │
    ├── Push Image
    │
    ▼
Docker / Kubernetes
    │
    ▼
Prometheus
    │
    ▼
Grafana
```

---

# ✅ 21. Final Checklist

| Component | Status to Verify |
|---|---|
| Ubuntu 24.04 | ⬜ |
| Ansible Controller | ⬜ |
| SSH Passwordless Authentication | ⬜ |
| `hosts` Inventory | ⬜ |
| Basic Packages | ⬜ |
| Docker Server | ⬜ |
| Monitoring Server | ⬜ |
| Prometheus | ⬜ |
| Grafana | ⬜ |
| Java 21 | ⬜ |
| PostgreSQL | ⬜ |
| SonarQube | ⬜ |
| Jenkins | ⬜ |
| Maven | ⬜ |
| Git | ⬜ |
| GCP Firewall | ⬜ |
| End-to-End Verification | ⬜ |

---

## 🎯 Lab Completion

When all checks pass, the environment is ready for the next phase:

**GitHub → Jenkins → Maven → SonarQube → Docker → Deployment → Prometheus → Grafana**

> **Tip:** Keep this file in the root of your GitHub project as the main infrastructure runbook. Each installation playbook can remain in the same repository and be executed independently.
