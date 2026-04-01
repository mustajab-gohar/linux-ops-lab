# Week 01: System Baseline & Security Hardening

## Scenario
Before exposing any server to a network, a strict security baseline must be established. This lab demonstrates the initial hardening phase of a Linux environment, adhering to the principles of least privilege, cryptographic authentication, and perimeter defense.

## Executed Tasks

### 1. Identity & Access Management (IAM)
Operating as the `root` user violates standard security practices. A dedicated operational user was created to handle daily administrative tasks.

**Commands Executed:**
- Create user with a dedicated home directory and zsh shell, and add to sudo group
```bash
sudo useradd systemops -m -s /usr/bin/zsh -G sudo
sudo passwd systemops
```

### 2. Cryptographic Authentication & SSH Hardening
Passwords are vulnerable to brute-force attacks. Access was migrated strictly to Ed25519 cryptographic keys, and the `.ssh` directory was locked down to satisfy OpenSSH strict mode requirements.

**Commands Executed:**
* Generated key pair locally (Client-side)
```bash
ssh-keygen -t ed25519
```

* Secured the `.ssh` directory and `authorized_keys` file (Server-side)
```bash
sudo mkdir /home/systemops/.ssh
sudo chmod 700 /home/systemops/.ssh
touch /home/systemops/.ssh/authorized_keys
sudo chmod 600 /home/systemops/.ssh/authorized_keys
```

**sshd Configuration File Modifications:**
To finalize the lockdown, the SSH daemon was reconfigured to reject unauthorized access methods:
- Changed to: `PermitRootLogin no` (Explicitly disabled all remote root access).
- Changed to: `PasswordAuthentication no` (Disabled password fallback, enforcing key-only access).

**Service Restart:**
```bash
sudo systemctl restart ssh
```

### 3. Verification & Log Auditing
To verify the SSH hardening was successful, a deliberate attempt was made to log in as `root`. The authentication log output below confirms the daemon successfully dropped the connection at the `[preauth]` stage.

**Log Evidence:**
```text
Apr 01 19:55:42 kali sshd-session[251797]: Connection reset by authenticating user root 192.168.56.1 port 54083 [preauth]
Apr 01 19:55:42 kali sshd-session[251797]: PAM 1 more authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=192.168.56.1  user=root
Apr 01 19:55:44 kali sshd-session[252140]: Connection reset by authenticating user root 192.168.56.1 port 54632 [preauth]
```

### 4. Network Perimeter Defense (UFW)
Established a default-drop perimeter using the Uncomplicated Firewall (UFW) to hide all unused ports from network scanners.

**Commands Executed:**
```bash
sudo ufw allow 22 comment 'SSH Port'
sudo ufw enable
sudo ufw status
```
