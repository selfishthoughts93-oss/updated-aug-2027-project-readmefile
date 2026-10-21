# GCP DevOps Ansible Runbook — august2027

This runbook uses the existing GCP VM layout and Ansible YAML playbooks. The SonarQube playbook uses the official SonarQube Community Docker image instead of downloading the ZIP from `binaries.sonarsource.com`, because that endpoint returned HTTP 403 in the user's environment. This avoids the failing ZIP download URL; it does not guarantee Docker Hub access if outbound access is blocked.

## 1. VM inventory

| Group | VM | Internal IP | SSH user |
|---|---|---:|---|
| `jenkins` | `august2027-jenkins-vm` | `10.10.0.2` | Jenkins controller uses local Ansible connection |
| `docker` | `august2027-docker-vm` | `10.10.0.3` | `docker` |
| `monitoring` | `august2027-monitoring-vm` | `10.10.0.4` | `monitoring` |
| `sonarqube` | `august2027-sonarqube-vm` | `10.10.0.5` | `sonar` |

Jenkins controller shell user shown in the terminal: `rushiawslabs`. SSH key path used by the inventory: `/home/rushiawslabs/.ssh/id_ed25519_gcp`.

## 2. Set up passwordless SSH from Jenkins to each managed server

Run these commands on the Jenkins VM as `rushiawslabs` (do not use `sudo` for the SSH key commands):

```bash
whoami
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

If you already have a private/public key pair at the configured path, reuse it. Otherwise create one:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_gcp -C "jenkins-to-august2027-vms"
```

For unattended Ansible use, press Enter twice for an empty passphrase. Keep the private key (`id_ed25519_gcp`) only on Jenkins. Never paste or copy the private key to another VM.

Display the public key and copy the complete single line:

```bash
cat ~/.ssh/id_ed25519_gcp.pub
```

### Install the public key manually on each destination VM

In Google Cloud Console, open SSH-in-browser individually for each VM. Log in as the intended SSH account and verify it with `whoami`. Run:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
```

Paste the Jenkins **public** key as one complete new line, preserving existing lines. Save (`Ctrl+O`, Enter, `Ctrl+X`) and run:

```bash
chmod 600 ~/.ssh/authorized_keys
```

If the account's home directory or ownership is wrong, correct it as an administrator. `authorized_keys` must belong to the same Linux account named by `ansible_user`.

Repeat for:

- Docker: account `docker`, VM `10.10.0.3`
- Monitoring: account `monitoring`, VM `10.10.0.4`
- SonarQube: account `sonar`, VM `10.10.0.5`

The accounts must exist and the SSH server must be running on each destination. Ensure GCP firewall rules allow TCP/22 from the Jenkins VM or its private subnet. Prefer internal VPC IPs for this traffic.

## 3. Avoid interactive first-time fingerprint prompts safely

On Jenkins, create/edit `~/.ssh/config`:

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

Then:

```bash
chmod 600 ~/.ssh/config
```

`accept-new` saves a previously unseen host key without asking `yes/no`, but rejects a changed host key. Verify each host fingerprint through a trusted channel before first use. Do not use `StrictHostKeyChecking no` as a routine workaround.

## 4. Test SSH before running Ansible

Run on Jenkins:

```bash
ssh -o BatchMode=yes docker hostname
ssh -o BatchMode=yes monitoring hostname
ssh -o BatchMode=yes sonarqube hostname
```

Each should print the destination hostname without asking for the destination account password. If not, use verbose diagnostics:

```bash
ssh -vvv docker
```

Check username, key path, `authorized_keys` ownership and permissions, SSH service, and GCP firewall before continuing.

## 5. Ansible controller prerequisites

From the Jenkins VM:

```bash
sudo apt update
sudo apt install -y ansible python3-docker
ansible-galaxy collection install community.docker
```

Put the supplied files in `~/ansible`:

- `hosts`
- `Basic-Packages-install.yml`
- `Docker-install.yml`
- `Monitoring-install.yml`
- `SonarQube-install.yml`
- `Jenkins-install.yml`

Review the inventory key path and usernames against the actual VM accounts. Check inventory parsing and reachability:

```bash
cd ~/ansible
ansible-inventory -i hosts --graph
ansible docker,monitoring,sonarqube -i hosts -m ping
```

## 6. SonarQube playbook change and important prerequisites

The SonarQube playbook installs Docker and PostgreSQL on the SonarQube VM, creates the database and role, sets kernel limits, pulls `sonarqube:community` from Docker Hub, and runs the container with host networking so it can reach PostgreSQL on `127.0.0.1:5432`. Persistent data is stored under `/opt/sonarqube`.

Before running, edit `SonarQube-install.yml` and replace `CHANGE_ME_STRONG_PASSWORD` with a strong password. Prefer Ansible Vault for secrets. Do not commit a real password to source control.

The playbook requires the `community.docker` collection on the controller and outbound access from the SonarQube VM to the configured container registry. If Docker Hub is inaccessible, the image pull will fail and must be resolved at the registry/network level.

Validate and run only the SonarQube playbook:

```bash
ansible-playbook -i hosts SonarQube-install.yml --syntax-check
ansible-playbook -i hosts SonarQube-install.yml --limit sonarqube
```

Verify container and service endpoint:

```bash
ansible sonarqube -i hosts -m command -a "docker ps --filter name=sonarqube"
ansible sonarqube -i hosts -m uri -a "url=http://127.0.0.1:9000/api/system/status status_code=200"
```

Open `http://10.10.0.5:9000` only if your GCP firewall allows access from your trusted client IP. Avoid exposing SonarQube to the entire internet.

## 7. Run the other playbooks

```bash
ansible-playbook -i hosts Basic-Packages-install.yml --syntax-check
ansible-playbook -i hosts Docker-install.yml --syntax-check
ansible-playbook -i hosts Monitoring-install.yml --syntax-check
ansible-playbook -i hosts Jenkins-install.yml --syntax-check
```

Run each after reviewing its target group and settings. The Jenkins playbook uses `connection: local`; run it only if you intend to install/configure Jenkins on the controller VM.

## 8. Security notes

- Never copy a private SSH key to Docker, Monitoring, or SonarQube.
- Passwordless SSH does not mean disabling host-key validation.
- `accept-new` avoids first-connection prompts but is not a substitute for verifying fingerprints.
- Use Ansible Vault for database passwords.
- Restrict SSH and web ports in GCP firewall rules to trusted sources.
- The supplied SonarQube playbook uses the mutable `sonarqube:community` image tag. For repeatable production deployments, pin a verified image version/digest after confirming the desired release.
