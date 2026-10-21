# GCP DevOps Infrastructure — Complete Ansible Runbook

**Project:** `august2027`  
**Zone:** `asia-south1-c`  
**Network:** `august2027-vpc`  
**Controller:** Jenkins VM, running Ansible as Linux user `rushiawslabs`  
**OS assumption:** Ubuntu servers; verify the actual release on each VM before execution.

This document consolidates the complete inventory, SSH passwordless setup, fingerprint handling, prerequisites, all installation playbooks, execution commands, verification and troubleshooting. It uses the Ansible YAML approach. SonarQube is deployed from the `sonarqube:community` Docker image to avoid the previously failing SonarSource ZIP endpoint that returned HTTP 403.

> **Important:** Commands and configuration are tailored to the IP addresses shown in your current GCP VM screenshot. Run the syntax checks first. The playbooks have not been executed against your live VMs as part of producing this document. Review passwords and firewall exposure before production use.

---

## 1. Current GCP VM details

| VM / role | Internal IP | External IP shown | Ansible group / access |
|---|---:|---:|---|
| Jenkins / Ansible controller | `10.10.0.2` | `8.231.84.102` | `jenkins` uses local connection; Linux login shown as `rushiawslabs` |
| Docker | `10.10.0.3` | `35.200.140.236` | `docker`, SSH user `docker` |
| Monitoring | `10.10.0.4` | `34.47.135.87` | `monitoring`, SSH user `monitoring` |
| SonarQube | `10.10.0.5` | `34.47.164.139` | `sonarqube`, SSH user `sonar` |

Use internal IPs for traffic between VMs on the same VPC. Confirm that the actual Linux usernames exist on each VM. The Jenkins playbook targets `localhost` so it does not need SSH back into the Jenkins VM.

## 2. Prepare Jenkins / Ansible controller

Connect using the GCP Console **SSH** button for `august2027-jenkins-vm`. Confirm the account and home directory:

```bash
whoami
echo "$HOME"
# Expected from your current session: rushiawslabs and /home/rushiawslabs
```

Install the Ansible and SSH client tools:

```bash
sudo apt update
sudo apt install -y ansible openssh-client python3 python3-pip
ansible --version
ssh -V
```

Create the working directory and enter it:

```bash
mkdir -p ~/ansible
cd ~/ansible
```

Put the files from this runbook into `~/ansible` using your preferred editor or upload mechanism. The expected files are:

```text
~/ansible/
├── hosts
├── Basic-Packages-install.yml
├── Docker-install.yml
├── Monitoring-install.yml
├── SonarQube-install.yml
└── Jenkins-install.yml
```

## 3. Configure passwordless SSH from Jenkins to managed VMs

This configures one-way SSH from the Jenkins/Ansible controller to Docker, Monitoring and SonarQube. It does not automatically configure SSH in the reverse direction.

### 3.1 Check or create a key pair

Run on Jenkins as `rushiawslabs` (do not run key creation with `sudo`):

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
ls -l ~/.ssh/id_ed25519_gcp ~/.ssh/id_ed25519_gcp.pub
```

If both files exist and you trust the key, reuse them. Otherwise create the key:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_gcp -C "jenkins-ansible-august2027"
```

For non-interactive lab automation, leave the passphrase empty by pressing Enter twice. A passphrase-protected key is safer but requires an agent or another secure unlock mechanism.

Set permissions:

```bash
chmod 600 ~/.ssh/id_ed25519_gcp
chmod 644 ~/.ssh/id_ed25519_gcp.pub
```

Display **only the public key** to copy it manually:

```bash
cat ~/.ssh/id_ed25519_gcp.pub
```

The output is one line beginning with `ssh-ed25519`. Never display, paste, or copy `~/.ssh/id_ed25519_gcp` (the private key) to other servers.

### 3.2 Install the public key individually on each destination VM

For each VM, open its own SSH-in-browser session in Google Cloud Console. Log in as the account that Ansible will use, verify it with `whoami`, then install the public key in that account's `~/.ssh/authorized_keys` file.

On Docker VM (logged in as `docker`, if that is the correct login account):

```bash
whoami
echo "$HOME"
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
```

Paste the complete single public-key line copied from Jenkins. Do not remove existing authorized keys. Save (`Ctrl+O`, Enter, `Ctrl+X`) and run:

```bash
chmod 600 ~/.ssh/authorized_keys
```

Repeat the same steps individually in the Monitoring VM as `monitoring`, and in the SonarQube VM as `sonar`. Make sure the file is under the intended login account's actual home directory and owned by that account. If you edited the file as root, correct ownership using the verified account and home path, for example:

```bash
sudo chown -R docker:docker /home/docker/.ssh
```

Use the corresponding username and home directory on the other servers. Do not assume the home path if `echo "$HOME"` shows something different.

### 3.3 Configure SSH aliases and suppress first-use Yes/No prompts safely

On Jenkins:

```bash
nano ~/.ssh/config
```

Add this configuration. It uses `accept-new`, which automatically records previously unseen host keys but rejects changed keys:

```sshconfig
Host docker
    HostName 10.10.0.3
    User docker
    IdentityFile ~/.ssh/id_ed25519_gcp
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new

Host monitoring
    HostName 10.10.0.4
    User monitoring
    IdentityFile ~/.ssh/id_ed25519_gcp
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new

Host sonarqube
    HostName 10.10.0.5
    User sonar
    IdentityFile ~/.ssh/id_ed25519_gcp
    IdentitiesOnly yes
    StrictHostKeyChecking accept-new
```

Save and secure it:

```bash
chmod 600 ~/.ssh/config
```

**Security:** Before first connection, verify host fingerprints through a trusted channel if possible. Do not use `StrictHostKeyChecking no` as a blanket workaround. `accept-new` removes the interactive prompt for new hosts, but still protects against changed host keys.

### 3.4 Test each SSH connection from Jenkins

```bash
ssh docker hostname
ssh monitoring hostname
ssh sonarqube hostname
```

Each should log in without the destination account password or interactive host-key prompt once the key is installed and the host key is saved. If the first login asks to accept a fingerprint, verify the host identity before proceeding.

If SSH key authentication fails, use verbose output:

```bash
ssh -vvv docker
```

Check that the username matches the account whose `authorized_keys` contains the public key, that the private key is readable only by its owner, and that port 22 is reachable. In GCP, ensure an ingress firewall rule allows TCP port 22 from the Jenkins VM/internal subnet or an appropriately narrow source range.

## 4. Ansible inventory — `hosts`

Create `~/ansible/hosts` with the following content. The `jenkins` group is local because Jenkins itself is the controller; this avoids unnecessary SSH back to itself.

```ini
# GCP august2027 DevOps inventory
# Jenkins controller runs Ansible locally; use private VPC IPs for remote hosts.

[docker]
10.10.0.3 ansible_user=docker

[monitoring]
10.10.0.4 ansible_user=monitoring

[sonarqube]
10.10.0.5 ansible_user=sonar

[jenkins]
localhost ansible_connection=local ansible_python_interpreter=/usr/bin/python3

[all:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_private_key_file=/home/rushiawslabs/.ssh/id_ed25519_gcp
ansible_ssh_common_args='-o StrictHostKeyChecking=accept-new'
```

Check the parsed inventory:

```bash
cd ~/ansible
ansible-inventory -i hosts --graph
```

Test managed nodes:

```bash
ansible docker,monitoring,sonarqube -i hosts -m ping
```

Expected for each managed node: `SUCCESS` and `"ping": "pong"`. Ansible ping is a Python/SSH connectivity test, not an ICMP ping.

## 5. Install the required Docker collection

The Monitoring and SonarQube playbooks use `community.docker` modules. Install the collection on Jenkins:

```bash
ansible-galaxy collection install community.docker
ansible-galaxy collection list | grep community.docker
```

The remote servers must have Python 3. The playbooks install relevant packages where needed.

## 6. Basic packages playbook — `Basic-Packages-install.yml`

Create `~/ansible/Basic-Packages-install.yml`:

```yaml
---
- name: Configure common packages on remote DevOps servers
  hosts: docker,monitoring,sonarqube
  become: true
  gather_facts: true
  tasks:
    - name: Update apt package cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Install common packages
      ansible.builtin.apt:
        name:
          - ca-certificates
          - curl
          - git
          - wget
          - unzip
          - vim
          - zip
          - net-tools
          - htop
          - python3
        state: present

    - name: Display remote hostname
      ansible.builtin.command: hostname
      register: host_name
      changed_when: false

    - name: Show remote hostname
      ansible.builtin.debug:
        var: host_name.stdout
```

Validate and execute:

```bash
ansible-playbook -i hosts Basic-Packages-install.yml --syntax-check
ansible-playbook -i hosts Basic-Packages-install.yml
```

## 7. Docker installation playbook — `Docker-install.yml`

Create `~/ansible/Docker-install.yml`:

```yaml
---
- name: Install and configure Docker Engine on Docker server
  hosts: docker
  become: true
  gather_facts: true
  tasks:
    - name: Update apt package cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Install Ubuntu Docker package
      ansible.builtin.apt:
        name:
          - docker.io
          - ca-certificates
        state: present

    - name: Enable and start Docker
      ansible.builtin.service:
        name: docker
        state: started
        enabled: true

    - name: Add remote login user to docker group when it exists
      ansible.builtin.user:
        name: "{{ ansible_user }}"
        groups: docker
        append: true
      when: ansible_user is defined

    - name: Verify Docker version
      ansible.builtin.command: docker --version
      register: docker_version
      changed_when: false

    - name: Show Docker version
      ansible.builtin.debug:
        var: docker_version.stdout
```

Validate, execute and verify:

```bash
ansible-playbook -i hosts Docker-install.yml --syntax-check
ansible-playbook -i hosts Docker-install.yml
ansible docker -i hosts -b -m command -a "docker --version"
ansible docker -i hosts -b -m command -a "docker ps"
```

## 8. Monitoring playbook — `Monitoring-install.yml`

This deploys Prometheus on port 9090 and Grafana on port 3000 using Docker containers. The supplied playbook does not mount a custom Prometheus configuration or persistent data volumes; add those if required for your monitoring design.

Create `~/ansible/Monitoring-install.yml`:

```yaml
---
- name: Install Prometheus and Grafana containers on Monitoring server
  hosts: monitoring
  become: true
  gather_facts: true
  tasks:
    - name: Update apt package cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Install Docker Engine
      ansible.builtin.apt:
        name: docker.io
        state: present

    - name: Enable and start Docker
      ansible.builtin.service:
        name: docker
        state: started
        enabled: true

    - name: Ensure Prometheus container is present and running
      community.docker.docker_container:
        name: prometheus
        image: prom/prometheus:latest
        state: started
        restart_policy: unless-stopped
        published_ports:
          - "9090:9090"
      register: prometheus_container

    - name: Ensure Grafana container is present and running
      community.docker.docker_container:
        name: grafana
        image: grafana/grafana:latest
        state: started
        restart_policy: unless-stopped
        published_ports:
          - "3000:3000"
      register: grafana_container

    - name: Show monitoring container status
      ansible.builtin.command: docker ps
      register: container_status
      changed_when: false

    - name: Display monitoring containers
      ansible.builtin.debug:
        var: container_status.stdout_lines
```

Validate and execute:

```bash
ansible-playbook -i hosts Monitoring-install.yml --syntax-check
ansible-playbook -i hosts Monitoring-install.yml
ansible monitoring -i hosts -b -m command -a "docker ps"
```

Web endpoints from a permitted network:

- Prometheus: `http://34.47.135.87:9090`
- Grafana: `http://34.47.135.87:3000`

Restrict GCP firewall exposure to trusted IP ranges; do not expose administrative dashboards publicly without suitable access controls.

## 9. SonarQube + PostgreSQL playbook — `SonarQube-install.yml`

This complete playbook installs Docker and PostgreSQL on the SonarQube VM, creates the database and role, configures the kernel limits, **pulls the SonarQube Community image, creates the `sonarqube` container, starts it, and waits for the status API to report `UP`**. It uses host networking so the container can reach PostgreSQL on the VM at `127.0.0.1:5432`; no SonarSource ZIP download is used, avoiding the previous HTTP 403 download endpoint.

**Before running:** replace `CHANGE_ME_STRONG_PASSWORD` with a strong PostgreSQL password. Prefer Ansible Vault for secrets. Avoid single quotes in the placeholder password if using the SQL tasks below, or adapt the SQL quoting to your secret-management policy. This playbook includes `no_log` on the task that handles the password.

If the container was missing because an older YAML file was being run, replace the entire file on Jenkins with the code below. The key tasks to confirm are **Pull SonarQube Community image from Docker Hub** and **Create and start SonarQube container**.

Create or replace `~/ansible/SonarQube-install.yml` with this complete code:

```yaml
---
- name: Install and configure SonarQube Community with PostgreSQL using Docker
  hosts: sonarqube
  become: true
  gather_facts: true

  vars:
    postgres_db: sonarqube
    postgres_user: sonar
    # Replace this placeholder before running. Prefer Ansible Vault for secrets.
    postgres_password: "CHANGE_ME_STRONG_PASSWORD"
    sonar_image: "sonarqube:community"
    sonar_data_dir: /opt/sonarqube/data
    sonar_extensions_dir: /opt/sonarqube/extensions
    sonar_logs_dir: /opt/sonarqube/logs

  tasks:
    - name: Confirm target is Ubuntu
      ansible.builtin.assert:
        that:
          - ansible_distribution == "Ubuntu"
        fail_msg: "This playbook is intended for Ubuntu servers."

    - name: Update APT cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Install Docker, PostgreSQL and required packages
      ansible.builtin.apt:
        name:
          - docker.io
          - postgresql
          - postgresql-contrib
          - python3-docker
          - ca-certificates
          - curl
          - procps
          - unzip
        state: present

    - name: Enable and start Docker
      ansible.builtin.service:
        name: docker
        state: started
        enabled: true

    - name: Enable and start PostgreSQL
      ansible.builtin.service:
        name: postgresql
        state: started
        enabled: true

    - name: Create or update SonarQube PostgreSQL role
      ansible.builtin.shell: |
        set -euo pipefail
        if runuser -u postgres -- psql -tAc "SELECT 1 FROM pg_roles WHERE rolname='{{ postgres_user }}'" | grep -q 1; then
          runuser -u postgres -- psql -v ON_ERROR_STOP=1 -c "ALTER USER {{ postgres_user }} WITH PASSWORD '{{ postgres_password }}';"
        else
          runuser -u postgres -- psql -v ON_ERROR_STOP=1 -c "CREATE USER {{ postgres_user }} WITH PASSWORD '{{ postgres_password }}';"
        fi
      args:
        executable: /bin/bash
      no_log: true
      changed_when: true

    - name: Create SonarQube database if missing
      ansible.builtin.shell: |
        set -euo pipefail
        if ! runuser -u postgres -- psql -tAc "SELECT 1 FROM pg_database WHERE datname='{{ postgres_db }}'" | grep -q 1; then
          runuser -u postgres -- createdb -O {{ postgres_user }} {{ postgres_db }}
          echo created
        else
          echo exists
        fi
      args:
        executable: /bin/bash
      register: sonar_db_result
      changed_when: "'created' in sonar_db_result.stdout"

    - name: Configure kernel limits required by SonarQube Elasticsearch
      ansible.builtin.copy:
        dest: /etc/sysctl.d/99-sonarqube.conf
        owner: root
        group: root
        mode: "0644"
        content: |
          vm.max_map_count=524288
          fs.file-max=131072
      register: sonar_sysctl_file

    - name: Apply kernel limits
      ansible.builtin.command:
        argv:
          - sysctl
          - --system
      when: sonar_sysctl_file.changed
      changed_when: true

    - name: Create persistent SonarQube directories
      ansible.builtin.file:
        path: "{{ item }}"
        state: directory
        owner: "1000"
        group: "1000"
        mode: "0755"
      loop:
        - "{{ sonar_data_dir }}"
        - "{{ sonar_extensions_dir }}"
        - "{{ sonar_logs_dir }}"

    - name: Pull SonarQube Community image from Docker Hub
      community.docker.docker_image:
        name: "{{ sonar_image }}"
        source: pull

    - name: Create and start SonarQube container
      community.docker.docker_container:
        name: sonarqube
        image: "{{ sonar_image }}"
        state: started
        restart_policy: unless-stopped
        network_mode: host
        env:
          SONAR_JDBC_URL: "jdbc:postgresql://127.0.0.1:5432/{{ postgres_db }}"
          SONAR_JDBC_USERNAME: "{{ postgres_user }}"
          SONAR_JDBC_PASSWORD: "{{ postgres_password }}"
        volumes:
          - "{{ sonar_data_dir }}:/opt/sonarqube/data"
          - "{{ sonar_extensions_dir }}:/opt/sonarqube/extensions"
          - "{{ sonar_logs_dir }}:/opt/sonarqube/logs"
        ulimits:
          - "nofile:131072:131072"
          - "nproc:8192:8192"
      no_log: true

    - name: Wait for SonarQube to report UP
      ansible.builtin.uri:
        url: http://127.0.0.1:9000/api/system/status
        method: GET
        return_content: true
        status_code:
          - 200
          - 503
      register: sonar_http_status
      retries: 36
      delay: 10
      until:
        - sonar_http_status.status == 200
        - sonar_http_status.json.status is defined
        - sonar_http_status.json.status == "UP"

    - name: Show SonarQube status and URL
      ansible.builtin.debug:
        msg:
          - "SonarQube is reporting UP."
          - "Internal URL: http://10.10.0.5:9000"
          - "Container name: sonarqube"
          - "Image: {{ sonar_image }}"
```

Install the required collection on Jenkins, then validate and run:

```bash
cd ~/ansible
ansible-galaxy collection install community.docker
ansible-playbook -i hosts SonarQube-install.yml --syntax-check
ansible-playbook -i hosts SonarQube-install.yml --limit sonarqube
```

Verify the result:

```bash
ansible sonarqube -i hosts -b -m command -a "docker ps -a"
ansible sonarqube -i hosts -b -m command -a "docker logs --tail 100 sonarqube"
ansible sonarqube -i hosts -m uri -a "url=http://127.0.0.1:9000/api/system/status return_content=true"
```

The expected container name is `sonarqube`. The image pull requires outbound access to Docker Hub. SonarQube may take several minutes on first startup. Internal URL: `http://10.10.0.5:9000`. Restrict TCP port 9000 in the GCP firewall to trusted sources.

> **If `docker ps -a` shows no `sonarqube` container after a successful play recap, confirm that you replaced the entire playbook and ran `ansible-playbook -i hosts SonarQube-install.yml --limit sonarqube` from the correct `~/ansible` directory. The earlier task list in your screenshot ended at kernel limits, which means the playbook being executed did not include the image-pull and container-start tasks shown here.**


## 10. Jenkins installation playbook — `Jenkins-install.yml`

This playbook installs Java 21, Maven, Docker, and Jenkins on the Jenkins VM using the official Jenkins Debian repository. It runs locally on the controller via the `localhost` inventory entry. The updated key URL uses the 2026 key filename to address the `NO_PUBKEY 7198F47B14ABC68` error seen in your terminal, and validates the downloaded key with GPG before adding the repository.

Create or replace `~/ansible/Jenkins-install.yml` with this complete code:

```yaml
---
- name: Install Jenkins prerequisites and Jenkins on controller
  hosts: jenkins
  connection: local
  become: true
  gather_facts: true

  vars:
    jenkins_repo_key_url: "https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key"
    jenkins_repo_key_path: /etc/apt/keyrings/jenkins-keyring.asc

  tasks:
    - name: Update APT cache before installing prerequisites
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Install Java 21 and required utilities
      ansible.builtin.apt:
        name:
          - fontconfig
          - openjdk-21-jre
          - curl
          - ca-certificates
          - gnupg
          - git
          - maven
          - docker.io
        state: present
        update_cache: true

    - name: Ensure APT keyrings directory exists
      ansible.builtin.file:
        path: /etc/apt/keyrings
        state: directory
        owner: root
        group: root
        mode: "0755"

    - name: Download current Jenkins repository signing key
      ansible.builtin.get_url:
        url: "{{ jenkins_repo_key_url }}"
        dest: "{{ jenkins_repo_key_path }}"
        owner: root
        group: root
        mode: "0644"
        force: true
        validate_certs: true
      register: jenkins_key_download

    - name: Verify downloaded Jenkins signing key is readable by GPG
      ansible.builtin.command:
        argv:
          - gpg
          - --batch
          - --show-keys
          - "{{ jenkins_repo_key_path }}"
      changed_when: false

    - name: Configure Jenkins APT repository with dedicated signing key
      ansible.builtin.apt_repository:
        repo: "deb [signed-by={{ jenkins_repo_key_path }}] https://pkg.jenkins.io/debian-stable binary/"
        filename: jenkins
        state: present
        update_cache: false

    - name: Refresh APT cache after configuring Jenkins repository
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 0

    - name: Install Jenkins package
      ansible.builtin.apt:
        name: jenkins
        state: present

    - name: Enable and start Docker
      ansible.builtin.service:
        name: docker
        state: started
        enabled: true

    - name: Add Jenkins service account to Docker group when account exists
      ansible.builtin.user:
        name: jenkins
        groups: docker
        append: true

    - name: Enable and start Jenkins
      ansible.builtin.service:
        name: jenkins
        state: started
        enabled: true

    - name: Wait for Jenkins HTTP port to become reachable locally
      ansible.builtin.wait_for:
        host: 127.0.0.1
        port: 8080
        delay: 5
        timeout: 180

    - name: Verify Jenkins service state
      ansible.builtin.command:
        argv:
          - systemctl
          - is-active
          - jenkins
      register: jenkins_status
      changed_when: false

    - name: Display Jenkins service status
      ansible.builtin.debug:
        msg: "Jenkins service status: {{ jenkins_status.stdout }}"
```

Validate and execute:

```bash
cd ~/ansible
ansible-playbook -i hosts Jenkins-install.yml --syntax-check
ansible-playbook -i hosts Jenkins-install.yml --limit jenkins
```

Then verify on the Jenkins VM:

```bash
sudo systemctl status jenkins --no-pager
sudo systemctl status docker --no-pager
java -version
mvn -version
sudo journalctl -u jenkins -n 100 --no-pager
```

Jenkins UI: `http://8.231.84.102:8080`, provided GCP firewall rules allow access from your trusted client IP. Do not expose port 8080 to the entire internet.

> **Signing-key note:** The key URL can change when Jenkins rotates repository signing keys. If the `jenkins.io-2026.key` URL returns 404 or APT still reports `NO_PUBKEY`, do not disable signature verification or use `trusted=yes`. Check the current key URL and fingerprint in the official [Jenkins Linux installation instructions](https://www.jenkins.io/doc/book/installing/linux/), update `jenkins_repo_key_url`, and rerun the playbook.


## 11. Run all playbooks in order

After SSH connectivity and the required collection are confirmed, run:

```bash
cd ~/ansible
ansible-playbook -i hosts Basic-Packages-install.yml
ansible-playbook -i hosts Docker-install.yml
ansible-playbook -i hosts Monitoring-install.yml
ansible-playbook -i hosts Jenkins-install.yml --limit jenkins
ansible-playbook -i hosts SonarQube-install.yml --limit sonarqube
```

If Jenkins and Ansible are already installed, you may not need to rerun `Jenkins-install.yml`. These playbooks are intended to be repeatable, but review any task that removes/recreates resources before rerunning it.

## 12. Final verification

```bash
ansible all -i hosts --list-hosts
ansible docker,monitoring,sonarqube -i hosts -m ping
ansible docker -i hosts -b -m command -a "docker ps"
ansible monitoring -i hosts -b -m command -a "docker ps"
ansible sonarqube -i hosts -b -m command -a "docker ps"
ansible jenkins -i hosts -b -m command -a "systemctl is-active jenkins"
```

For SonarQube, also check its status endpoint from the SonarQube VM:

```bash
curl -fsS http://127.0.0.1:9000/api/system/status
```

## 13. Troubleshooting quick reference

| Error | Check / action |
|---|---|
| `Permission denied (publickey)` | Correct SSH username, key path, destination `authorized_keys`, ownership and permissions. Test `ssh -vvv docker`. |
| `Connection timed out` on port 22 | GCP VPC firewall ingress, VM status, internal IP, route/network, and `sudo systemctl status ssh` on destination. |
| Fingerprint confirmation on first connection | Confirm host identity, then connect; `accept-new` stores unseen keys automatically. |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | Do not blindly delete the old key. Verify the VM was rebuilt and confirm the new fingerprint, then remove the stale entry with `ssh-keygen -R 10.10.0.X`. |
| Ansible `UNREACHABLE` | Test direct SSH first, then inspect inventory group/user/key path. |
| `community.docker` module missing | Run `ansible-galaxy collection install community.docker` on Jenkins. |
| Docker Hub pull failure | Check outbound internet/DNS and Docker daemon status on the target. |
| SonarQube returns 403 from the ZIP endpoint | The updated playbook uses the Docker image and does not download the SonarSource ZIP. Confirm the playbook contains the image-pull and container-start tasks; if Docker Hub is blocked, fix outbound access or configure an approved registry mirror. |
| SonarQube container not found | Run `ansible sonarqube -i hosts -b -m command -a "docker ps -a"`; replace the whole `SonarQube-install.yml` with the current version in this runbook and rerun it. |
| Jenkins `NO_PUBKEY` / repository not signed | Verify the current official Jenkins signing-key URL and fingerprint. Do not disable APT signature checking or set `trusted=yes`. |
| SonarQube container starts but does not become healthy | Inspect `docker logs sonarqube`, memory availability, `vm.max_map_count`, database credentials, and PostgreSQL status. |
| `sudo` password requested by Ansible | Run with `-K` to prompt for a sudo password, or configure a limited, policy-approved sudo rule. Avoid unrestricted `NOPASSWD: ALL` unless explicitly appropriate. |

## 14. Security notes

- Never copy the private SSH key (`id_ed25519_gcp`) to managed servers. Only install the `.pub` public key.
- Keep `~/.ssh` mode `700`, private key mode `600`, `authorized_keys` mode `600`, and SSH config mode `600`.
- `StrictHostKeyChecking accept-new` suppresses first-use prompts but rejects changed keys; it does not replace verifying host identity.
- Use internal IPs and restrict GCP firewall rules to required source ranges and ports.
- Store database credentials with Ansible Vault or a secret manager; do not commit secrets to source control.
- Docker group membership effectively grants root-level control over the Docker host. Grant it only to trusted accounts.
- Review exposure of Jenkins (8080), Grafana (3000), Prometheus (9090), and SonarQube (9000). Restrict access to trusted networks.

---

**End of runbook.**
