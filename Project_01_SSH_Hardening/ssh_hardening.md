## SSH Hardening Reference Guide
A comprehensive, security-oriented project to harden and monitor SSH access on Linux systems (tested on Fedora). This includes configuration, key management, firewall/SELinux integration, and log analysis.
### 1. What is SSH?

**SSH (Secure Shell)** is a cryptographic network protocol for securely operating network services over an unsecured network. It's mainly used for remote login and command execution.

#### Protocol Layers:

SSH operates at the **application layer** but provides services at the transport and network layers:

* **Transport Layer Protocol (TCP/22 or custom port):** Handles encryption, integrity, and authentication.
* **User Authentication Protocol:** Verifies the identity of the client.
* **Connection Protocol:** Multiplexes the encrypted tunnel into multiple logical channels (shell, forwarding, etc.).

---

### 2. Authentication Mechanisms in SSH

SSH supports multiple authentication types:

#### A. **Password-based Authentication**

* Server checks the password sent by client against system's shadow file.
* Vulnerable to brute-force unless restricted.

#### B. **Public Key Authentication**

* Most secure method.
* Workflow:

  * Client generates a key pair.
  * Public key is uploaded to server (`~/.ssh/authorized_keys`).
  * During login, server challenges client to prove it owns the private key.

#### C. **Host-based Authentication**

* Uses `.shosts` or `/etc/ssh/shosts.equiv`.
* Less common due to trust model.

#### D. **Keyboard-Interactive / PAM**

* Allows for multifactor setups like OTP.

#### Cryptographic Algorithms:

* **Ciphers**: AES-256-CTR, ChaCha20
* **MACs**: hmac-sha2-512, hmac-sha2-256
* **Kex**: curve25519-sha256, diffie-hellman-group-exchange-sha256

---

### 3. Common SSH Configuration Options

Edit file: `/etc/ssh/sshd_config`

| Option                                          | Description                      | Security Impact             |
| ----------------------------------------------- | -------------------------------- | --------------------------- |
| Port 4545                                       | Changes default port from 22     | Reduces noise from scanners |
| PermitRootLogin no                              | Disables root SSH login          | Critical                    |
| PasswordAuthentication no                       | Enforces key-only login          | High                        |
| PubkeyAuthentication yes                        | Enables public key auth          | High                        |
| MaxAuthTries 3                                  | Limits brute-force attempts      | Medium                      |
| ClientAliveInterval 100 / ClientAliveCountMax 2 | Drops idle sessions              | Medium                      |
| AllowUsers user1 user2                          | Restricts login to certain users | High                        |
| Protocol 2                                      | Enforces SSH v2 only             | Critical                    |

---

### 4. Custom Configuration File

Create a new file (e.g., `/etc/ssh/sshd_config.secure`) and populate it with:

```ini
Port 4545
Protocol 2
PermitRootLogin no
StrictModes yes
MaxAuthTries 3
MaxSessions 2
LoginGraceTime 30
PubkeyAuthentication yes
PasswordAuthentication no
PermitEmptyPasswords no
AllowUsers secssh ahmed
ClientAliveInterval 100
ClientAliveCountMax 2
Banner /etc/issue.net
Ciphers aes256-ctr,aes192-ctr,aes128-ctr
MACs hmac-sha2-512,hmac-sha2-256
KexAlgorithms curve25519-sha256@libssh.org,diffie-hellman-group-exchange-sha256
```

Then, restart the service:

```bash
sudo systemctl restart sshd
```

---

### 5. SSH Key Generation and Usage

```bash
# Generate key pair
ssh-keygen -t ed25519 -C "your_email@example.com"

# Copy key to server
ssh-copy-id -o Port=4545 user@host

# Connect via key
ssh -p 4545 user@host
```

---

### 6. SELinux and Firewall Configuration

#### FirewallD

```bash
firewall-cmd --add-port=4545/tcp --permanent
firewall-cmd --reload
```

#### SELinux

```bash
# Check if port 4545 is allowed
semanage port -l | grep ssh

# Add custom port if needed
semanage port -a -t ssh_port_t -p tcp 4545
```

---

### 7. Advanced Hardening Recommendations

* Restrict by user: `AllowUsers`, `AllowGroups`
* Use strong key exchange: `Ciphers`, `MACs`, `KexAlgorithms`
* Enable 2FA via Google Authenticator
* Enable Fail2Ban or auditd for brute-force protection
* Rate limit SSH using iptables:

```bash
iptables -A INPUT -p tcp --dport 4545 -m recent --set
iptables -A INPUT -p tcp --dport 4545 -m recent --update --seconds 60 --hitcount 3 -j DROP
```

* VPN-only access to SSH (e.g., via WireGuard)
* Run SSH in isolated containers/namespaces for attack surface minimization

---

### 8. SSH Banner (Legal Warning)

```bash
echo "Unauthorized access prohibited. All activities monitored." > /etc/issue.net
```

Add to sshd\_config:

```bash
Banner /etc/issue.net
```

---

### 9. Redirect SSH Logs to Custom File

By default, SSH logs go to syslog or journald. To redirect:

#### A. Rsyslog Method

Edit `/etc/rsyslog.conf` or `/etc/rsyslog.d/ssh.conf`:

```conf
if $programname == 'sshd' then /var/log/ssh_custom.log
& stop
```

Then:

```bash
sudo systemctl restart rsyslog
```

---

### 10. Analyzing SSH Logs

#### Recommended Tools:

| Tool                     | Use                              |
| ------------------------ | -------------------------------- |
| `grep`, `awk`, `sed`     | Text search and field extraction |
| `ausearch`, `auditd`     | Low-level monitoring             |
| `GoAccess`               | Real-time log visualization      |
| `Logwatch`               | Summary reports                  |
| `fail2ban-client status` | Banned IPs based on SSH logs     |
| `Python / Pandas`        | Structured analysis & filtering  |

#### Example: Extract Failed Attempts Using `awk`

```bash
awk '/Failed password/ {print $1, $2, $3, $11}' /var/log/ssh_custom.log
```

#### Python Example (pandas):

```python
import pandas as pd
log_lines = open("/var/log/ssh_custom.log").readlines()
failed = [l for l in log_lines if "Failed password" in l]
for line in failed:
    print(line.split())
```

---

### ✅ Summary

By combining layered security principles, enforcing cryptographic standards, isolating SSH traffic, monitoring logs, and applying user-based restrictions, your SSH service can be hardened against most attack vectors while remaining manageable and auditable.

Always monitor updates to OpenSSH and rotate keys regularly.
