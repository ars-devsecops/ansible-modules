# 🖥️ master-slave-software-install

> Ansible project to copy nginx offline packages from master → slave, install and start nginx — **zero internet required on slave server**
> by [@ars-devsecops](https://github.com/ars-devsecops)

![Ansible](https://img.shields.io/badge/Ansible-EE0000?style=flat-square&logo=ansible&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-009639?style=flat-square&logo=nginx&logoColor=white)
![Amazon EC2](https://img.shields.io/badge/Amazon_EC2-FF9900?style=flat-square&logo=amazonec2&logoColor=white)
![Amazon Linux](https://img.shields.io/badge/Amazon_Linux_2023-FF9900?style=flat-square&logo=amazonaws&logoColor=white)
![Status](https://img.shields.io/badge/Status-Production_Ready-00e5a0?style=flat-square)

---

## 📦 What This Repo Does

| Playbook | Description | Target |
|----------|-------------|--------|
| [setup-nginx-slave.yml](./playbooks/setup-nginx-slave.yml) | Copies nginx tar from master, installs offline, starts service | Slave server |

---

## 🏗️ Architecture

```
  MASTER SERVER                            SLAVE SERVER
  (Ansible Control Node)                   172.31.21.0 — private subnet
 ┌──────────────────────────┐             ┌──────────────────────────────┐
 │                          │             │                              │
 │  /tmp/                   │  SSH  ───▶  │  /tmp/nginx-packages.tar.gz  │
 │    nginx-packages.tar.gz │  (slave-    │  /tmp/nginx-packages/*.rpm   │
 │                          │   user)     │                              │
 │  setup-nginx-slave.yml   │             │  ✅ nginx installed          │
 │                          │             │  ✅ nginx started + enabled  │
 │  master-server-kp.pem    │             │  ✅ /tmp cleaned up          │
 └──────────────────────────┘             └──────────────────────────────┘
```

---

## 📁 Project Structure

```
master-slave-software-install/
│
├── playbooks/
│   └── setup-nginx-slave.yml     ← Main playbook
│
├── inventory/
│   └── hosts.ini                 ← Slave server IP + SSH config
│
├── files/
│   └── (place nginx-packages.tar.gz here — excluded from Git)
│
├── .gitignore
└── README.md
```

---

## ✅ Prerequisites

> Complete these steps **before** running the playbook.
> Split into two parts — Slave setup (one-time manual) and Master setup.

---

### 🔴 Part 1 — Slave Server Setup (One-time Manual)

SSH into the slave server as `ec2-user`:

```bash
ssh -i master-server-kp.pem ec2-user@172.31.21.0
```

#### Step 1 — Create `slave-user`
```bash
sudo useradd -m -s /bin/bash slave-user
sudo passwd slave-user
# Enter: Password@123
```

#### Step 2 — Set up `.ssh` directory for `slave-user`
```bash
sudo mkdir -p /home/slave-user/.ssh
sudo chmod 700 /home/slave-user/.ssh
sudo chown slave-user:slave-user /home/slave-user/.ssh
```

#### Step 3 — Copy `authorized_keys` from `ec2-user` → `slave-user`
```bash
sudo cp /home/ec2-user/.ssh/authorized_keys \
        /home/slave-user/.ssh/authorized_keys

sudo chown slave-user:slave-user /home/slave-user/.ssh/authorized_keys
sudo chmod 600 /home/slave-user/.ssh/authorized_keys
```

#### Step 4 — Enable password auth in `sshd_config`
```bash
sudo vi /etc/ssh/sshd_config
```

Find and update these two lines:
```
PasswordAuthentication yes
ChallengeResponseAuthentication yes
```

#### Step 5 — Restart SSH
```bash
sudo systemctl restart sshd
sudo systemctl status sshd
```

#### Step 6 — Verify `slave-user` can SSH from master
Run this from the **master server**:
```bash
ssh -i /home/ec2-user/master-server-kp.pem slave-user@172.31.21.0
# Should connect without password prompt
```

---

### 🟡 Part 2 — Master Server Setup

SSH into the master server and run:

#### Step 1 — Download nginx RPMs (internet needed only on master)
```bash
# Create download directory
mkdir -p /tmp/nginx-packages

# Download nginx + all dependencies
sudo dnf install nginx --downloadonly --downloaddir=/tmp/nginx-packages -y
sudo dnf download --resolve --alldeps nginx --downloaddir=/tmp/nginx-packages

# Verify RPMs downloaded
ls /tmp/nginx-packages
```

#### Step 2 — Package RPMs into tar.gz
```bash
cd /tmp
tar -czvf nginx-packages.tar.gz nginx-packages/

# Verify size
ls -lh /tmp/nginx-packages.tar.gz
```

#### Step 3 — Clone this repo on master
```bash
git clone https://github.com/ars-devsecops/master-slave-software-install.git
cd master-slave-software-install
```

#### Step 4 — Verify PEM key permissions
```bash
ls -l /home/ec2-user/master-server-kp.pem

# Fix permissions if needed
chmod 400 /home/ec2-user/master-server-kp.pem
```

#### Step 5 — Test master → slave connectivity
```bash
ansible slave -i inventory/hosts.ini -m ping
```

Expected output:
```
172.31.21.0 | SUCCESS => {
    "ping": "pong"
}
```

---

## 🚀 Running the Playbook

### Dry run first (no changes made)
```bash
ansible-playbook playbooks/setup-nginx-slave.yml \
  -i inventory/hosts.ini \
  --check -v
```

### Actual run
```bash
ansible-playbook playbooks/setup-nginx-slave.yml \
  -i inventory/hosts.ini \
  -v
```

---

## ▶️ What the Playbook Does — Step by Step

```
Step 1 → Pre-flight check    Verifies nginx-packages.tar.gz exists on master
Step 2 → Create dir          Creates /tmp/nginx-packages/ on slave
Step 3 → Copy tar            Copies tar from master → slave /tmp/
Step 4 → Extract             Unpacks all RPMs on slave
Step 5 → Install offline     dnf install *.rpm --disablerepo=* (no internet)
Step 6 → Start nginx         Starts + enables nginx via systemd
Step 7 → Health check        Verifies nginx ActiveState = active
Step 8 → Cleanup             Removes tar + RPMs from slave /tmp/
```

---

## ✅ Verify After Running

SSH into slave and confirm:

```bash
# Check nginx is running
systemctl status nginx

# Test nginx responds locally
curl http://localhost
# Returns: nginx welcome page HTML ✅

# Check enabled on boot
systemctl is-enabled nginx
```

Test from master:
```bash
curl http://172.31.21.0
```

---

## 🔧 Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `Permission denied (publickey)` | Wrong PEM or permissions | `chmod 400 *.pem` |
| `UNREACHABLE` | Security group blocking SSH | Allow port 22 from master IP in slave SG |
| `tar not found on master` | Skipped download step | Re-run `dnf download` + `tar` steps |
| `dnf install failed` | Missing dependency RPMs | Re-run with `--resolve --alldeps` |
| `nginx failed to start` | Port 80 already in use | `sudo fuser -k 80/tcp` then retry |
| `authorized_keys not working` | Wrong file permissions | `chmod 600 authorized_keys` + `chmod 700 .ssh` |

---

## 🔐 Security Notes

- ✅ `master-server-kp.pem` — excluded from Git via `.gitignore`
- ✅ `nginx-packages.tar.gz` — binary file, excluded from Git
- ⚠️ Change `slave-user` password from `Password@123` before production
- ⚠️ Consider disabling `PasswordAuthentication` after setup and using keys only

---

## 🤝 Author

**Amol Shinde** · Cloud & DevOps Engineer · AWS Security Specialist
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://linkedin.com/in/amol-shinde-profile)
[![GitHub](https://img.shields.io/badge/GitHub-ars--devsecops-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/ars-devsecops)
