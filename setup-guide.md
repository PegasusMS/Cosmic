# PegasusMS — Private Server Setup Guide

## Important caveat on "latest version"

There is **no working private server emulator for modern GMS** (2023+). The emulator community targets older protocol versions. The most realistic options are:

| Version | Project | State |
|---|---|---|
| **v83** | HeavenMS / Cosmic | Most stable, best documented |
| **v186–v214** | Various forks | Exist but less complete |
| **v225+** | Nothing reliable | Not publicly emulated |

For a first server, **v83 is the recommendation** — it has all the classic content (pre-Big Bang), active community, and the most documentation. If you want to push toward v186+, that's possible but expect rough edges.

---

## Where to get server software and client files

- **[Cosmic](https://github.com/P0nk/Cosmic)** — modern v83 fork, actively maintained (recommended)
- **[HeavenMS](https://github.com/ronancpl/HeavenMS)** — the original most-used v83 base
- **RaGEZONE** (`ragezone.com`) — the main private server forum; source for older WZ client files, v186+ server releases, and patching tools
- For v186+ server software, search RaGEZONE for the specific version

### Why Java?

All MapleStory private server emulators descend from **odinMS**, a Java-based emulator that leaked around 2007. Nexon's own server infrastructure was Java at the time, and odinMS was a reverse-engineered replication of it — also in Java. Every major server since (HeavenMS, Cosmic, etc.) is a fork of that lineage, so Java has just stuck around for two decades.

---

## Architecture

```
Your LAN
  ├── Beelink (Proxmox)
  │     └── LXC Container (e.g. 192.168.1.50)
  │           ├── Cosmic Java server
  │           └── MariaDB
  ├── Your PC  → client → 192.168.1.50
  └── Friend's PC → client → 192.168.1.50
```

---

## Step 1 — Create a Proxmox LXC container

1. In Proxmox web UI, download a **Ubuntu 22.04 LTS** CT template
2. Create a new LXC:
   - **CPU:** 2 cores
   - **RAM:** 3072 MB
   - **Disk:** 20 GB
   - **Network:** bridge `vmbr0`, static LAN IP (e.g. `192.168.1.50`)
3. Start it and update:

```bash
apt update && apt upgrade -y
```

---

## Step 2 — Install dependencies

```bash
apt install -y openjdk-8-jdk mariadb-server git maven
mysql_secure_installation
```

---

## Step 3 — Set up the database

```bash
mysql -u root -p
```

```sql
CREATE DATABASE heavenms;
CREATE USER 'maplestory'@'localhost' IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON heavenms.* TO 'maplestory'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Then import the schema (included in the repo):

```bash
mysql -u maplestory -p heavenms < /root/Cosmic/sql/database.sql
```

---

## Step 4 — Clone and configure the server

```bash
git clone https://github.com/P0nk/Cosmic.git
cd Cosmic
```

In the config file (`config.yaml` or the constants file), set:
- `HOST` → your container's LAN IP (e.g. `192.168.1.50`)
- DB credentials to match Step 3
- `ENABLE_PIN` / `ENABLE_PIC` → `false` for easier local play

Build:

```bash
mvn clean package -DskipTests
```

---

## Step 5 — Run persistently with systemd

Create `/etc/systemd/system/pegasusms.service`:

```ini
[Unit]
Description=PegasusMS MapleStory Server
After=network.target mariadb.service

[Service]
User=root
WorkingDirectory=/root/Cosmic
ExecStart=/usr/bin/java -jar target/Cosmic.jar
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable pegasusms
systemctl start pegasusms
```

---

## Step 6 — Client setup

1. Download the **v83 GMS client** (WZ files) from RaGEZONE
2. Add an entry to each player's Windows `hosts` file (`C:\Windows\System32\drivers\etc\hosts`):
   ```
   192.168.1.50   maplestory.nexon.net
   ```
3. Or use a client IP patcher tool (available on RaGEZONE)

---

## Step 7 — Firewall / ports

```bash
ufw allow 8484
ufw allow 7575:7576/tcp
```

| Port | Purpose |
|---|---|
| `8484` | Login server |
| `7575–7576` | Channel servers |
| `3306` | MariaDB (LAN only — do NOT expose externally) |

---

## Useful links

- Cosmic: https://github.com/P0nk/Cosmic
- HeavenMS: https://github.com/ronancpl/HeavenMS
- RaGEZONE MapleStory section: https://www.ragezone.com/forums/maple-story.19/
