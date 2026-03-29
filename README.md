# Building a Multi-Server Home Lab on AWS with Advanced Network Segmentation

In this project I built a multi-server infrastructure on AWS that replicates a real enterprise network. I deployed 5 EC2 instances across 4 network segments (subnets), each with a dedicated role, all connected inside a private VPC with layered security using both Security Groups and Network ACLs. A Bastion Host acts as the single secure entry point to the entire environment.

---

## What I Built

| Component | Description |
| --- | --- |
| Bastion Host | Amazon Linux 2023 — SSH gateway into the private network |
| Web Server | Ubuntu 22.04 — Apache HTTP server |
| Email Server | Ubuntu 22.04 — Postfix/Dovecot mail server |
| DNS Server | Ubuntu 22.04 — BIND9 recursive resolver |
| Database Server | Ubuntu 22.04 — MySQL database |
| Network | VPC with 4 subnets, NAT Gateway, NACLs, per-subnet route tables |

<img width="1408" height="768" alt="Gemini_Generated_Image_skjckmskjckmskjc" src="https://github.com/user-attachments/assets/808cc198-99aa-49f4-8480-fcc52497388b" />




---

## What I Did

### Step 1 — Created the VPC and Subnets

I started by building the network before launching any machines. I created a VPC named `Multi-Server-vpc` with a CIDR block of `10.0.0.0/16`, then created four subnets to segment the network by role:

| Subnet | CIDR | Purpose |
| --- | --- | --- |
| PublicSubnet | 10.0.1.0/24 | Internet-facing entry point |
| DMZSubnet | 10.0.2.0/24 | Semi-public services (Web, Email) |
| AppSubnet | 10.0.3.0/24 | Internal services (DNS) |
| DataSubnet | 10.0.4.0/24 | Sensitive data storage (Database) |
<img width="1613" height="318" alt="image" src="https://github.com/user-attachments/assets/8d6e2bd9-aa3c-40c7-99c5-85461b5b3554" />


I then created an Internet Gateway named `Multi-Server-GW` and attached it to the VPC so the public subnet could reach the internet.
<img width="1607" height="190" alt="image" src="https://github.com/user-attachments/assets/5ec053a1-ee6a-4dca-a18f-9f3d38e2ee53" />

---

### Step 2 — Set Up NAT Gateway and Route Tables

I created a NAT Gateway named `Multi-Server-NAT` inside the PublicSubnet so the private servers could reach the internet for updates without being directly accessible from the outside. I allocated an Elastic IP and assigned it to the NAT Gateway.
<img width="1605" height="191" alt="image" src="https://github.com/user-attachments/assets/36746c4d-7e4d-428e-921a-0ddf0e2103e0" />


I then created four separate route tables — one per subnet — to control traffic flow:

- **PublicRT** → default route points to the Internet Gateway
- **DMZRT** → default route points to the NAT Gateway
- **AppRT** → default route points to the NAT Gateway
- **DataRT** → default route points to the NAT Gateway
<img width="1597" height="322" alt="image" src="https://github.com/user-attachments/assets/83458975-76d7-495f-a5a3-6c9794b91819" />


Each route table was associated with its respective subnet.

---

### Step 3 — Created Security Groups

I created a Security Group for each server role to act as instance-level firewalls:

- **SG-Bastion** — SSH (port 22) from my public IP only
- **SG-Web** — HTTP/HTTPS from anywhere, SSH from SG-Bastion only
- **SG-Email** — SMTP from anywhere, IMAP from VPC, SSH from SG-Bastion only
- **SG-DNS** — DNS (UDP/TCP 53) from the entire VPC, SSH from SG-Bastion only
- **SG-DB** — MySQL (port 3306) from DMZ subnet only, SSH from SG-Bastion only
<img width="1597" height="382" alt="image" src="https://github.com/user-attachments/assets/3e522e83-40b0-4d1a-9355-bc44aca95b41" />

---

### Step 4 — Created Network ACLs (Subnet-Level Firewalls)

On top of Security Groups, I added Network ACLs for defense in depth. Security Groups are like locks on each room door — NACLs are locks on each floor's entrance. Even if someone gets past one layer, they still face the other.

I created three custom NACLs:

- **NACL-DMZ** — Allows HTTP, HTTPS, SMTP, and SSH from the public subnet inbound
- **NACL-App** — Allows DNS (UDP/TCP 53) from the VPC and SSH from the public subnet inbound
- **NACL-Data** — Allows MySQL from the DMZ subnet and SSH from the public subnet inbound
<img width="1611" height="347" alt="image" src="https://github.com/user-attachments/assets/66538481-f248-4fe7-acbf-d20930094863" />

Each NACL included ephemeral port rules (1024–65535) for return traffic since NACLs are stateless.

> **Troubleshooting: NACLs are stateless — this tripped me up multiple times.** The original NACL rules only allowed TCP ephemeral ports, but DNS uses UDP. This caused DNS queries to time out because UDP return traffic was being silently dropped. I had to add Custom UDP rules for ports 1024–65535 on both the App and DMZ subnet NACLs.
<img width="1585" height="750" alt="image" src="https://github.com/user-attachments/assets/acfd2d66-7af8-44e3-8ab6-da5d105717c4" />
<img width="1597" height="751" alt="image" src="https://github.com/user-attachments/assets/4dad81f9-2ebd-4a07-886a-a24ae2fab1ae" />


---

### Step 5 — Launched EC2 Instances

I launched all 5 servers through EC2, assigning each to its designated subnet and security group. I set static private IPs manually in the advanced network settings during launch (under "Primary IP"):

- Bastion Host → 10.0.1.10 (PublicSubnet, auto-assign public IP enabled)
- Web Server → 10.0.2.12 (DMZSubnet)
- Email Server → 10.0.2.14 (DMZSubnet)
- DNS Server → 10.0.3.11 (AppSubnet)
- DB Server → 10.0.4.13 (DataSubnet)
<img width="1602" height="373" alt="image" src="https://github.com/user-attachments/assets/286e7294-6161-485c-a299-d3fbb07ad790" />


After launching the Bastion Host, I allocated an Elastic IP and associated it so I'd always have a consistent public address to SSH into.
<img width="1583" height="752" alt="image" src="https://github.com/user-attachments/assets/e78e25d7-7d18-4c4d-8636-74976f4976f6" />

---

### Step 6 — Connected via SSH (Bastion Jump Host)

Since the private servers have no public IP, I had to SSH through the Bastion Host. I connected to the Bastion from my local PowerShell:

```bash
ssh -i C:\Users\yamen\Downloads\MultiServerKeyPair.pem ec2-user@<BASTION_ELASTIC_IP>
```

> **Troubleshooting: Key file not found.** My first attempt failed with `Identity file MultiServerKeyPair.pem not accessible: No such file or directory` because I didn't provide the full Windows path to the `.pem` file. Always use the full path like `C:\Users\yamen\Downloads\MultiServerKeyPair.pem`.

To hop to private servers, I initially tried SSH-ing into the Bastion first, then running another SSH command from there. This failed with `Permission denied (publickey)` because the key file path `C:\Users\...` doesn't exist on the Bastion — that's a Windows path and the Bastion is Linux.

> **Troubleshooting: ProxyCommand solved the jump host problem.** Instead of two-step SSH, I used a single command from my local PowerShell that tunnels through the Bastion while keeping the key local:
>
> ```bash
> ssh -i C:\Users\yamen\Downloads\MultiServerKeyPair.pem \
>   -o ProxyCommand="ssh -i C:\Users\yamen\Downloads\MultiServerKeyPair.pem -W %h:%p ec2-user@<BASTION_IP>" \
>   ubuntu@<PRIVATE_IP>
> ```

I used this pattern for all private servers, just changing the target IP:
- Web Server → `ubuntu@10.0.2.12`
- Email Server → `ubuntu@10.0.2.14`
- DNS Server → `ubuntu@10.0.3.11`
- DB Server → `ubuntu@10.0.4.13`

---

### Step 7 — Configured the DNS Server (BIND9)

I SSH'd into the DNS Server and installed BIND9:

```bash
sudo apt update && sudo apt install -y bind9 bind9utils
```

I edited `/etc/bind/named.conf.options` and added forwarders, query restrictions, and recursion inside the `options { }` block:

```
forwarders { 8.8.8.8; 8.8.4.4; };
allow-query { 10.0.0.0/16; localhost; };
recursion yes;

<img width="913" height="701" alt="image" src="https://github.com/user-attachments/assets/5cb77fba-7503-472e-ac7f-54f9a2b754e5" />

```

> **Troubleshooting: BIND failed to start — missing semicolons.** My first attempt to restart BIND failed. Running `sudo named-checkconf` showed `missing ';' before 'allow-query'`. The issue was that I forgot the trailing semicolon after the closing brace on the `forwarders` line. Every block in BIND config needs `};` not just `}`.
<img width="923" height="693" alt="image" src="https://github.com/user-attachments/assets/36491e56-5ba6-47b2-914e-ac051447252e" />

> **Troubleshooting: `systemctl enable bind9` refused.** The error said `Refusing to operate on alias name`. The actual service name on Ubuntu is `named`, not `bind9`. I used `sudo systemctl enable named` instead.

> **Troubleshooting: BIND couldn't resolve — IPv6 network unreachable.** After restarting, `systemctl status named` showed `network unreachable resolving 'google.com'` on IPv6 addresses, and the resolver priming query timed out. The VPC doesn't support IPv6, so BIND was trying to reach root servers over a protocol that wasn't available. I fixed this by adding `listen-on-v6 { none; };` to the BIND config.
<img width="823" height="176" alt="image" src="https://github.com/user-attachments/assets/8828927b-14bf-4d42-80d9-01bef09063f3" />

> **Troubleshooting: Typo — "non" instead of "none".** After saving the config, `named-checkconf` returned `undefined ACL 'non'`. I had written `listen-on-v6 { non; };` instead of `listen-on-v6 { none; };`. A one-letter typo that took a minute to spot.
<img width="830" height="172" alt="image" src="https://github.com/user-attachments/assets/6a29a7cb-402b-4c4e-bc39-850499ee4909" />

> **Troubleshooting: DNS still timing out even after IPv6 fix.** Running `dig @8.8.8.8 google.com` from the DNS Server itself timed out. The problem was the App Subnet NACL — it only had TCP ephemeral ports allowed inbound, but DNS responses come back on UDP. Adding a Custom UDP rule for ports 1024–65535 on NACL-App fixed it.

> **Troubleshooting: Web Server couldn't query DNS across subnets.** Even after DNS worked locally on the DNS Server, `nslookup google.com 10.0.3.11` from the Web Server still timed out. Same root cause — the DMZ Subnet NACL was also missing a UDP ephemeral ports rule. Adding it to NACL-DMZ resolved the issue.

---

### Step 8 — Configured the Web Server (Apache)

I SSH'd into the Web Server and installed Apache:

```bash
sudo apt update && sudo apt install -y apache2
sudo systemctl start apache2
sudo systemctl enable apache2
echo "<h1>HomeLabWeb is running!</h1>" | sudo tee /var/www/html/index.html
```

---

### Step 9 — Configured the Database Server (MySQL)

I SSH'd into the DB Server and installed MySQL:

```bash
sudo apt update && sudo apt install -y mysql-server
sudo systemctl start mysql
sudo systemctl enable mysql
sudo mysql_secure_installation
```

During secure installation, I chose MEDIUM password validation and answered `y` to all security prompts. Then I created a database and a user restricted to the Web Server's IP:

```sql
CREATE DATABASE homelab_db;
CREATE USER 'labuser'@'10.0.2.12' IDENTIFIED BY 'SecurePass123!';
GRANT ALL PRIVILEGES ON homelab_db.* TO 'labuser'@'10.0.2.12';
FLUSH PRIVILEGES;
```

> **Troubleshooting: Web Server couldn't connect to MySQL.** From the Web Server, `mysql -h 10.0.4.13 -u labuser -p homelab_db` returned `Can't connect to MySQL server on '10.0.4.13:3306' (111)`. The issue was that MySQL was bound to `127.0.0.1` (localhost only), rejecting all remote connections. I edited `/etc/mysql/mysql.conf.d/mysqld.cnf`, changed `bind-address` from `127.0.0.1` to `0.0.0.0`, and restarted MySQL with `sudo systemctl restart mysql`.
<img width="902" height="697" alt="image" src="https://github.com/user-attachments/assets/8dc4237b-87d7-4ad3-8de6-1e4bf56eaf8e" />

> **Note:** I also had to install the MySQL client on the Web Server first since it wasn't included by default: `sudo apt install -y mysql-client-core-8.0`

---

### Step 10 — Configured the Email Server (Postfix/Dovecot)

I SSH'd into the Email Server and installed Postfix and Dovecot:

```bash
sudo apt update && sudo apt install -y postfix dovecot-core dovecot-imapd
sudo systemctl start postfix dovecot
sudo systemctl enable postfix dovecot
```

During Postfix setup, I chose "Internet Site" and set the mail name to `homelab.local`. I also had to install `mailutils` since the `mail` command wasn't available by default:

```bash
sudo apt install -y mailutils
```

I tested by sending a local email and verifying delivery:

```bash
echo 'Test email from HomeLabVM' | mail -s 'Test' root@homelab.local
sudo cat /var/mail/root
```
<img width="926" height="166" alt="image" src="https://github.com/user-attachments/assets/ebdb04d7-3976-482a-bd03-998468674188" />

---

### Step 11 — Tested Connectivity Across Subnets

From the Bastion Host, I verified that all servers were reachable:

```bash
ping -c 3 10.0.2.12   # Web Server (DMZ)
<img width="791" height="261" alt="Web Server (DMZ)" src="https://github.com/user-attachments/assets/b0e28324-49ee-42c7-8941-07c89e49155b" />

ping -c 3 10.0.2.14   # Email Server (DMZ)
<img width="827" height="165" alt="Email Server (DMZ)" src="https://github.com/user-attachments/assets/6ce30e8b-e2fd-4c9d-98e7-9dd2bbc95b7f" />

ping -c 3 10.0.3.11   # DNS Server (App Tier)
<img width="803" height="164" alt="DNS Server (App Tier)" src="https://github.com/user-attachments/assets/12f590c1-df63-4b5d-ad51-2994b723e02e" />

ping -c 3 10.0.4.13   # DB Server (Data Tier)
<img width="782" height="170" alt="DB Server (Data Tier)" src="https://github.com/user-attachments/assets/eacdfe4a-8ec2-4bf6-aed9-2f2d71c7327d" />

```

I also tested cross-subnet service connectivity:
- DNS resolution from the Web Server: `nslookup google.com 10.0.3.11` ✓
- <img width="737" height="695" alt="image" src="https://github.com/user-attachments/assets/32dd3650-d9b3-43f5-b390-d774a30feb0a" />

- MySQL connection from the Web Server: `mysql -h 10.0.4.13 -u labuser -p homelab_db` ✓
- <img width="913" height="378" alt="image" src="https://github.com/user-attachments/assets/e7695d73-e253-4aac-9b19-2e08f7e3fbce" />

- HTTP from the DB Server: `curl http://10.0.2.12` ✓
<img width="925" height="392" alt="image" src="https://github.com/user-attachments/assets/4ae8938a-30d7-471e-ae20-7459e0d500b1" />

---

### Step 12 — Verified Network Segmentation

I confirmed that unauthorized cross-subnet traffic was properly blocked by the NACLs:

- From DB Server → Email Server: `curl http://10.0.2.14` — timed out ✓
- <img width="1087" height="92" alt="From DB Server, try to reach Email Server directly (should fail)" src="https://github.com/user-attachments/assets/8a495e53-4597-4b6b-8130-492e8a098b07" />

- From DNS Server → MySQL: `mysql -h 10.0.4.13 -u labuser -p` — refused ✓
- <img width="846" height="86" alt="From DNS Server, try to connect to MySQL (should fail)" src="https://github.com/user-attachments/assets/fce11b0b-ef82-4b93-b063-aae7d0348d23" />


This proves the segmentation is working. Even if the DMZ were compromised, the attacker would need to bypass the Data Tier NACL to reach the database.

---

## Key Takeaways

1. **NACLs are stateless** — you must explicitly allow return traffic (ephemeral ports 1024–65535) for BOTH TCP and UDP. This was my biggest pain point and caused multiple DNS failures.
2. **BIND config is semicolon-sensitive** — always run `sudo named-checkconf` before restarting to catch syntax errors early.
3. **MySQL defaults to localhost only** — if other servers need to connect remotely, change `bind-address` to `0.0.0.0`.
4. **SSH through a Bastion requires key management** — use `ProxyCommand` so the key stays on your local machine instead of trying to reference Windows paths from a Linux host.
5. **IPv6 can break DNS in a VPC** — if your VPC doesn't support IPv6, disable it in BIND with `listen-on-v6 { none; };`.
6. **Defense in depth works** — Security Groups + NACLs together create two independent layers. Even misconfiguring one doesn't leave you completely exposed.

---

## Technologies Used

- AWS (VPC, EC2, Security Groups, Network ACLs, NAT Gateway, Elastic IPs, Route Tables)
- BIND9 (DNS Server)
- Apache2 (Web Server)
- MySQL (Database Server)
- Postfix / Dovecot (Email Server)
- SSH with ProxyCommand (Bastion Host jumping)
- PowerShell on Windows (local terminal)
