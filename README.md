# 🛡️ Building a Home SIEM Lab with ELK Stack — Detecting SSH Brute Force Attacks

> A hands-on cybersecurity project: Deploy Elasticsearch, Logstash & Kibana using Docker, collect logs with Filebeat, simulate a real SSH brute-force attack from Kali Linux, and visualize the detections in Kibana.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Step 1 — Deploy the ELK Stack with Docker](#step-1--deploy-the-elk-stack-with-docker)
- [Step 2 — Install & Configure Filebeat](#step-2--install--configure-filebeat)
- [Step 3 — Simulate an SSH Brute Force Attack (Kali Linux)](#step-3--simulate-an-ssh-brute-force-attack-kali-linux)
- [Step 4 — Detect & Visualize in Kibana](#step-4--detect--visualize-in-kibana)
- [What You Learned](#what-you-learned)
- [Next Steps](#next-steps)

---

## Overview

A **SIEM (Security Information and Event Management)** system is one of the most important tools in a SOC analyst's toolkit. It collects logs from across an environment, centralizes them, and lets you search, alert on, and visualize security events in real time.

In this project, we build a fully functional mini-SIEM on a local Ubuntu VM using the **ELK Stack**:

| Component | Role |
|---|---|
| **Elasticsearch** | Stores and indexes all log data |
| **Kibana** | Visualizes and queries the data |
| **Filebeat** | Lightweight agent that ships logs to Elasticsearch |

We then attack the Ubuntu machine from **Kali Linux** using **Hydra** (SSH brute force) and watch the events appear in our SIEM.

---

## Architecture

```
┌─────────────────────────────────┐       ┌──────────────────────┐
│         Ubuntu VM               │       │      Kali Linux       │
│                                 │       │                       │
│  ┌──────────┐  ┌─────────────┐  │◄──────│  hydra (brute force) │
│  │Filebeat  │  │  Docker     │  │  SSH  │  ssh fakeuser@...    │
│  │(agent)   │  │  ┌────────┐ │  │       └──────────────────────┘
│  │          │  │  │Elastic │ │  │
│  │ /var/log │──►  │search  │ │  │
│  │ auth.log │  │  └────┬───┘ │  │
│  └──────────┘  │       │     │  │
│                │  ┌────▼───┐ │  │
│                │  │Kibana  │ │  │
│                │  │:5601   │ │  │
│                │  └────────┘ │  │
│                └─────────────┘  │
└─────────────────────────────────┘
```

---

## Prerequisites

- **Ubuntu VM** (22.04 LTS recommended) — VirtualBox or VMware
- **Kali Linux VM** (attacker machine) — on the same virtual network
- Minimum **4GB RAM** allocated to Ubuntu VM (ELK is memory-hungry)
- Both VMs should be able to **ping each other** (use Bridged or Host-Only adapter)

---

## Step 1 — Deploy the ELK Stack with Docker

### 1.1 — Update Ubuntu and Install Docker

SSH into your Ubuntu VM or open a terminal and run the following commands one at a time.

**Update system packages:**
```bash
sudo apt update && sudo apt upgrade -y
```

**Install Docker and Docker Compose:**
```bash
sudo apt install docker.io docker-compose-v2 -y
```

**Enable and start Docker:**
```bash
sudo systemctl enable docker
sudo systemctl start docker
```

**Verify Docker is working:**
```bash
docker --version
```

Expected output:
```
Docker version 24.x.x, build xxxxxxx
```

---

### 1.2 — Create the ELK Directory

```bash
mkdir ~/elk-stack
cd ~/elk-stack
```

---

### 1.3 — Create the Docker Compose File

```bash
nano docker-compose.yml
```

Paste the following configuration:

```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.13.4
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms2g -Xmx2g
    ports:
      - "9200:9200"
    networks:
      - elk

  kibana:
    image: docker.elastic.co/kibana/kibana:8.13.4
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch
    networks:
      - elk

networks:
  elk:
```

Save and exit nano:
- `CTRL + O` → Save
- `ENTER` → Confirm filename
- `CTRL + X` → Exit

---

### 1.4 — Start the ELK Stack

```bash
sudo docker compose up -d
```

The `-d` flag runs the containers in detached (background) mode. The first run will download the Docker images — this may take a few minutes depending on your internet speed.

**Check that both containers are running:**
```bash
sudo docker ps
```

You should see two containers listed:

```
CONTAINER ID   IMAGE                                      STATUS         PORTS
xxxxxxxxxxxx   docker.elastic.co/kibana/kibana:8.13.4    Up X minutes   0.0.0.0:5601->5601/tcp
xxxxxxxxxxxx   docker.elastic.co/elasticsearch/...       Up X minutes   0.0.0.0:9200->9200/tcp
```

---

### 1.5 — Verify the Stack is Working

**Access Kibana in your browser:**
```
http://localhost:5601
```

You should see the Kibana welcome/home screen.

**Test Elasticsearch via curl:**
```bash
curl http://localhost:9200
```

Expected JSON response:
```json
{
  "name" : "your-node-name",
  "cluster_name" : "docker-cluster",
  "version" : {
    "number" : "8.13.4",
    ...
  },
  "tagline" : "You Know, for Search"
}
```

✅ If you can see both — your ELK Stack is live.

---

## Step 2 — Install & Configure Filebeat

Filebeat is a lightweight log shipper that runs on the Ubuntu host (outside Docker). It reads system logs and forwards them directly to Elasticsearch.

### 2.1 — Download Filebeat

```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.13.4-amd64.deb
```

### 2.2 — Install Filebeat

```bash
sudo dpkg -i filebeat-8.13.4-amd64.deb
```

### 2.3 — Enable the System Logs Module

The system module tells Filebeat to collect `auth.log` and `syslog` — the exact logs that capture SSH login attempts.

```bash
sudo filebeat modules enable system
```

### 2.4 — Set Up Filebeat (load index templates into Elasticsearch)

```bash
sudo filebeat setup -e
```

> ⚠️ This may take a minute or two. It loads dashboards and index mappings into Elasticsearch and Kibana.

### 2.5 — Start Filebeat

```bash
sudo systemctl enable filebeat
sudo systemctl start filebeat
```

**Verify Filebeat is running:**
```bash
sudo systemctl status filebeat
```

You should see `Active: active (running)`.

---

## Step 3 — Simulate an SSH Brute Force Attack (Kali Linux)

Now we switch to our **Kali Linux** attacker machine and simulate a real-world attack — the kind a threat actor would run against an exposed SSH service.

> 🔴 **Important:** Only perform this against machines you own or have explicit permission to test. This is a lab environment — both machines are yours.

### 3.1 — Find Your Ubuntu VM's IP Address

On your **Ubuntu VM**, run:
```bash
ip a
```

Note the IP address (e.g., `192.168.1.105`). You'll use this in the commands below.

### 3.2 — Run a Brute Force Attack with Hydra

On your **Kali Linux** machine, run:

```bash
hydra -l testuser -P /usr/share/wordlists/rockyou.txt ssh://<your-ubuntu-ip> -t 4
```

Replace `<your-ubuntu-ip>` with the actual IP. 

What this does:
- `-l testuser` — tries the username `testuser`
- `-P /usr/share/wordlists/rockyou.txt` — uses the infamous rockyou wordlist as passwords
- `ssh://` — targets the SSH service
- `-t 4` — runs 4 parallel threads

> 💡 You don't need to let it finish. Let it run for 30–60 seconds to generate enough failed login events, then `CTRL + C` to stop it.

### 3.3 — Simulate Manual Failed SSH Logins

Also try some manual failed SSH attempts — these generate clean, readable auth.log entries:

```bash
ssh fakeuser@<your-ubuntu-ip>
```

Enter a wrong password a few times, then `CTRL + C`.

---

## Step 4 — Detect & Visualize in Kibana

### 4.1 — Open Kibana

In your browser on the Ubuntu VM:
```
http://localhost:5601
```

### 4.2 — Navigate to Discover

1. Click the **hamburger menu** (☰) in the top-left
2. Go to **Analytics → Discover**
3. Select the `filebeat-*` index pattern from the data view dropdown

### 4.3 — Search for Failed SSH Logins

In the search bar, enter:

```
system.auth.ssh.event: "Failed"
```

Or search for:
```
message: "Failed password"
```

You should see a flood of log entries generated by Hydra — each one representing a failed SSH login attempt.

### 4.4 — Key Fields to Examine

| Field | What It Shows |
|---|---|
| `system.auth.ssh.event` | `Failed` or `Accepted` |
| `source.ip` | IP address of the attacker (your Kali IP) |
| `user.name` | The username that was attempted |
| `@timestamp` | When the event occurred |
| `host.name` | The machine that received the attack |

### 4.5 — View Pre-built Dashboards

Filebeat ships with ready-made dashboards:

1. Go to **Analytics → Dashboards**
2. Search for `[Filebeat System]`
3. Open the **SSH login attempts** dashboard

You'll see graphs showing login attempt frequency, source IPs, and failed vs. successful logins over time.

---

## What You Learned

By completing this project, you have demonstrated the following skills:

| Skill | How It Was Applied |
|---|---|
| **Linux administration** | Installing packages, managing services with `systemctl` |
| **Docker & containerization** | Deploying multi-service stacks with Docker Compose |
| **Log management** | Configuring Filebeat to ship system logs |
| **SIEM fundamentals** | Centralized log collection, indexing, and search |
| **Threat detection** | Identifying brute-force patterns in auth logs |
| **Attack simulation** | Using Hydra to generate realistic attack traffic |
| **Security visualization** | Building and reading dashboards in Kibana |

---

## Next Steps

Here are ways to extend this project further:

- 🔔 **Set up Kibana Alerts** — Get notified when failed SSH logins exceed a threshold (e.g., 10 failures in 60 seconds)
- 🗺️ **Add GeoIP enrichment** — Map attacker IPs to geographic locations on a world map dashboard
- 🔒 **Enable Elasticsearch security** — Turn `xpack.security.enabled` back on and configure TLS + authentication
- 📦 **Add Logstash** — Insert Logstash between Filebeat and Elasticsearch for log enrichment and transformation pipelines
- 🖥️ **Ingest Windows logs** — Add a Windows VM and ship Windows Event Logs (Event ID 4625 = failed logon) into the same SIEM
- 🚨 **Integrate Wazuh** — Replace or complement Filebeat with Wazuh for host-based intrusion detection (HIDS) and built-in detection rules

---

## 📁 Project Structure

```
elk-stack/
└── docker-compose.yml     # ELK Stack service definitions
```

---

## 🧰 Tools & Versions Used

| Tool | Version |
|---|---|
| Ubuntu | 22.04 LTS |
| Docker | 24.x |
| Elasticsearch | 8.13.4 |
| Kibana | 8.13.4 |
| Filebeat | 8.13.4 |
| Kali Linux | Rolling |
| Hydra | Pre-installed on Kali |

---

## ⚠️ Disclaimer

This project is intended for **educational purposes only** in an isolated lab environment. All attack simulations were performed on machines owned and controlled by the author. Never run brute-force tools against systems you do not own or have explicit written permission to test.

---

*Built as part of a personal cybersecurity home lab series. If this helped you, consider starring the repo ⭐*
