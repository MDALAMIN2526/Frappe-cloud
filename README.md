Here’s a **step-by-step installation plan** to self-host **Frappe Cloud (Press Console)** across **two Ubuntu 20.04 servers**, where one acts as the **Press + Frappe App Server** and the other as the **Database (MariaDB) Server**, all under **Cloudflare Tunnel and DNS** for secure, private, and public access.

---

## 🌐 Overview

| Server       | Role           | Example Hostname    | Components                                |
| ------------ | -------------- | ------------------- | ----------------------------------------- |
| **Server 1** | Press + Frappe | `press.example.com` | Press Console, Frappe Sites, Redis, Nginx |
| **Server 2** | Database       | `db.internal`       | MariaDB, backups, private access only     |

---

## 🧱 Network Design

* **Private network:** WireGuard will connect the two servers privately (no public DB port).
* **Public access:** Cloudflare Tunnel exposes only the Press/Frappe web interface.
* **DNS:** Cloudflare DNS points to the Cloudflare Tunnel hostname.
* **Both servers are local**, so no direct public IPs needed.

---

## 🖥️ Step 1: Prepare Both Servers

Run on both servers:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl wget git unzip software-properties-common ufw -y
```

Enable basic firewall:

```bash
sudo ufw allow OpenSSH
sudo ufw enable
```

---

## 🔒 Step 2: Setup WireGuard Private Network

### On **Server 1 (Press + Frappe)**

```bash
sudo apt install wireguard -y
cd /etc/wireguard
umask 077
wg genkey | tee server1.key | wg pubkey > server1.pub
```

### On **Server 2 (DB Server)**

```bash
sudo apt install wireguard -y
cd /etc/wireguard
umask 077
wg genkey | tee server2.key | wg pubkey > server2.pub
```

Now, exchange the **public keys** (`server1.pub` and `server2.pub`) between the servers.

---

### Configure WireGuard

#### On **Server 1: `/etc/wireguard/wg0.conf`**

```
[Interface]
PrivateKey = <server1.key contents>
Address = 10.10.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <server2.pub>
AllowedIPs = 10.10.0.2/32
```

#### On **Server 2: `/etc/wireguard/wg0.conf`**

```
[Interface]
PrivateKey = <server2.key contents>
Address = 10.10.0.2/24

[Peer]
PublicKey = <server1.pub>
Endpoint = <SERVER1_PUBLIC_OR_TUNNEL_IP>:51820
AllowedIPs = 10.10.0.1/32
PersistentKeepalive = 25
```

Start and enable WireGuard:

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

Verify:

```bash
ping -c 4 10.10.0.2  # from server 1
ping -c 4 10.10.0.1  # from server 2
```

---

## 🧰 Step 3: Setup Database Server (Server 2)

Install MariaDB:

```bash
sudo apt install mariadb-server mariadb-client -y
sudo systemctl enable mariadb
sudo systemctl start mariadb
```

Secure it:

```bash
sudo mysql_secure_installation
```

When asked:

* Remove anonymous users: ✅
* Disallow root login remotely: ✅
* Remove test database: ✅
* Reload privilege tables: ✅

Create a **Frappe DB user** restricted to private WireGuard IP:

```bash
sudo mysql -u root -p
CREATE DATABASE frappe_db;
CREATE USER 'frappe_user'@'10.10.0.1' IDENTIFIED BY 'StrongPassword123!';
GRANT ALL PRIVILEGES ON frappe_db.* TO 'frappe_user'@'10.10.0.1';
FLUSH PRIVILEGES;
EXIT;
```

---

## ⚙️ Step 4: Setup Press + Frappe Server (Server 1)

### 1️⃣ Install Dependencies

```bash
sudo apt install python3-dev python3.8-venv python3-pip redis-server wkhtmltopdf supervisor nginx -y
```

Install Node.js (v18 or later):

```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
```

---

### 2️⃣ Setup Bench & Frappe

```bash
sudo useradd -m -s /bin/bash frappe
sudo su - frappe
cd ~
pip3 install frappe-bench
bench init press --frappe-branch version-15
cd press
```

---

### 3️⃣ Configure DB Connection (to private WireGuard IP)

Edit `sites/common_site_config.json`:

```json
{
  "db_host": "10.10.0.2",
  "db_port": 3306,
  "db_name": "frappe_db",
  "db_user": "frappe_user",
  "db_password": "StrongPassword123!"
}
```

---

### 4️⃣ Install Press Console

```bash
bench get-app https://github.com/frappe/press.git
bench new-site press.localhost
bench --site press.localhost install-app press
```

---

## 🧩 Step 5: Setup Cloudflare Tunnel (Server 1)

1. Install Cloudflare Tunnel:

   ```bash
   sudo mkdir -p /etc/cloudflared
   sudo cloudflared service install <YOUR_TUNNEL_TOKEN>
   ```

2. Configure tunnel route to `localhost:8000` (Bench default).

   `/etc/cloudflared/config.yml`:

   ```yaml
   tunnel: <YOUR_TUNNEL_ID>
   credentials-file: /etc/cloudflared/<YOUR_TUNNEL_ID>.json
   ingress:
     - hostname: press.example.com
       service: http://localhost:8000
     - service: http_status:404
   ```

3. Restart tunnel:

   ```bash
   sudo systemctl restart cloudflared
   ```

---

## 🧠 Step 6: Test and Validate

* Access your Press dashboard at your Cloudflare domain (e.g., `https://press.example.com`)
* Verify DB connection:

  ```bash
  mysql -h 10.10.0.2 -u frappe_user -p frappe_db
  ```
* Ensure both servers ping each other over `10.10.0.x`

---

## 🧾 Optional Step: Enable SSL via Cloudflare

* Enable **Full (strict)** SSL mode in Cloudflare dashboard
* No need for Let’s Encrypt locally (Cloudflare handles SSL)

---

## ⚡ Resource Recommendations

| Role           | CPU     | RAM   | Disk       | Users         |
| -------------- | ------- | ----- | ---------- | ------------- |
| Press + Frappe | 8 cores | 32 GB | 300 GB SSD | 200–250 users |
| DB Server      | 8 cores | 32 GB | 400 GB SSD | 300 users     |

If you plan to scale beyond 300 users, you can add a **third app node** later.

---


