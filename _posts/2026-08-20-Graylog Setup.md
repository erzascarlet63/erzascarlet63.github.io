---
title: Graylog Deployment via Docker 
date: 2026-08-20 00:00:00 +0800
categories: tools,siem 
tags: 
description: Setting up centralized monitoring system 
image: assets/posts/2026-08-20-Graylog/graylog1.png
---

# Graylog Open

**Environment:** Synology NAS (Container Manager / Docker)
**Graylog version:** 7.1 (with Graylog Data Node)
**Last updated:** August 2026

---

## 1. Overview

This blog describes the deployment of Graylog Open as a centralized syslog server for network switch monitoring, hosted via Docker on a Synology NAS using Container Manager.

**Stack components:**
- MongoDB 7.0 — Graylog's configuration/metadata database
- Graylog Data Node 7.1 — wraps OpenSearch, handles search/indexing
- Graylog Server 7.1 — main application and web UI

**Network design:** Graylog is exposed on the LAN via a **macvlan** network with a dedicated static IP, separate from the NAS's own IP. This avoids Docker NAT, which was found to strip the real source IP from incoming syslog UDP packets.

---

## 2. Prerequisites

- Synology NAS with **Container Manager** installed (DSM 7.2+)
- SSH access enabled (Control Panel → Terminal & SNMP)
- At least 4 GB RAM free (Data Node's OpenSearch component is memory-hungry)
- A free, confirmed-unused static IP address on the LAN for Graylog

### 2.1 Set `vm.max_map_count`

Required for the Data Node's OpenSearch process to start.

```bash
sudo sysctl -w vm.max_map_count=262144
```

Make it persistent across reboots via **Control Panel → Task Scheduler → Create → Triggered Task → User-defined script**, Event: **Boot-up**, run as root:
```bash
sysctl -w vm.max_map_count=262144
```

---

## 3. Project Files

Project folder: `/volume1/<username>/docker/graylog/`

### 3.1 `.env`

```
GRAYLOG_PASSWORD_SECRET=<96-character random string>
GRAYLOG_ROOT_PASSWORD_SHA2=<sha256 hash of your chosen admin password>
```

Generate the password secret:
```bash
< /dev/urandom tr -dc A-Z-a-z-0-9 | head -c96; echo
```

Generate the root password hash:
```bash
echo -n "yourpassword" | sha256sum
```

> **Note:** `.env` holds the raw secret values. `docker-compose.yml` references them via `${VARIABLE_NAME}` syntax — never edit the `${...}` lines in the compose file directly.

### 3.2 `docker-compose.yml`

Macvlan gives the Graylog container its own real identity on the LAN its own IP, its own MAC address, as if it were a separate physical device plugged directly into your network. No NAT, no translation layer. [DevOpsSchool](https://www.devopsschool.com/blog/how-to-assign-a-static-ip-to-a-docker-container-in-bridge-mode/)

```yaml
services:
  mongodb:
    image: "mongo:7.0"
    restart: "on-failure"
    networks:
      graylog:
        ipv4_address: "192.168.64.z"
    volumes:
      - "mongodb_data:/data/db"
      - "mongodb_config:/data/configdb"

  datanode:
    image: "${DATANODE_IMAGE:-graylog/graylog-datanode:7.1}"
    hostname: "datanode"
    environment:
      GRAYLOG_DATANODE_NODE_ID_FILE: "/var/lib/graylog-datanode/node-id"
      GRAYLOG_DATANODE_PASSWORD_SECRET: "${GRAYLOG_PASSWORD_SECRET:?Please configure GRAYLOG_PASSWORD_SECRET in the .env file}"
      GRAYLOG_DATANODE_MONGODB_URI: "mongodb://mongodb:27017/graylog"
      opensearch.bootstrap.system_call_filter: "false"
      GRAYLOG_DATANODE_OPENSEARCH_HEAP: "3g"
      JAVA_OPTS: "-Xms3g -Xmx3g"
    ulimits:
      memlock:
        hard: -1
        soft: -1
      nofile:
        soft: 65536
        hard: 65536
    ports:
      - "8999:8999/tcp"
      - "9200:9200/tcp"
      - "9300:9300/tcp"
    networks:
      graylog:
        ipv4_address: "192.168.64.y"
    volumes:
      - "graylog-datanode:/var/lib/graylog-datanode"
    restart: "on-failure"

  graylog:
    hostname: "server"
    image: "${GRAYLOG_IMAGE:-graylog/graylog:7.1}"
    depends_on:
      mongodb:
        condition: "service_started"
      datanode:
        condition: "service_started"
    entrypoint: "/usr/bin/tini -- /docker-entrypoint.sh"
    environment:
      GRAYLOG_NODE_ID_FILE: "/usr/share/graylog/data/data/node-id"
      GRAYLOG_PASSWORD_SECRET: "${GRAYLOG_PASSWORD_SECRET:?Please configure GRAYLOG_PASSWORD_SECRET in the .env file}"
      GRAYLOG_ROOT_PASSWORD_SHA2: "${GRAYLOG_ROOT_PASSWORD_SHA2:?Please configure GRAYLOG_ROOT_PASSWORD_SHA2 in the .env file}"
      GRAYLOG_HTTP_BIND_ADDRESS: "0.0.0.0:9000"
      GRAYLOG_HTTP_EXTERNAL_URI: "http://<ip site>:9000/"
      GRAYLOG_MONGODB_URI: "mongodb://mongodb:27017/graylog"
      GRAYLOG_ROOT_TIMEZONE: <YOUR TIMEZONE>
    networks:
      graylog:
        ipv4_address: "192.168.64.x"
      macvlan_net:
        ipv4_address: <ip site>
    volumes:
      - "graylog_data:/usr/share/graylog/data"
    restart: "on-failure"

networks:
  graylog:
    driver: "bridge"
    ipam:
      config:
        - subnet: "192.168.64.0/24"
  macvlan_net:
    driver: macvlan
    driver_opts:
      parent: ovs_eth0
    ipam:
      config:
        - subnet: <subnet>
          gateway: <gateway>

volumes:
  mongodb_data:
  mongodb_config:
  graylog-datanode:
  graylog_data:
```

**Key design decisions:**
- All internal (bridge) IPs are **statically pinned** (`192.168.64.x/y/z`). This prevents Docker from reassigning random bridge-network IPs on every container recreation, which was found to invalidate the Data Node's TLS certificate and break the indexer connection.
- Graylog itself sits on **two** networks: the internal bridge (to reach MongoDB/Data Node) and macvlan (to get a real, routable LAN identity).
- `mongodb` and `datanode` do **not** need macvlan — only Graylog needs to be directly reachable by switches and browsers.

---

## 4. Networking Notes (Synology-Specific)

### 4.1 Interface name

Synology uses **Open vSwitch**, so the physical interface is typically `ovs_eth0`, not `eth0`. Confirm with:
```bash
ip addr show | grep -E "^[0-9]+:"
```

### 4.2 Gateway

```bash
ip route | grep default
```

### 4.3 macvlan host-isolation limitation

**The NAS itself cannot reach its own macvlan-networked containers.** This is a Linux kernel/Docker design limitation, not a misconfiguration. All testing of the macvlan IP must be done from a **separate device** on the LAN, not via SSH/ping/curl from the NAS itself.

### 4.4 IP conflict checking

Before assigning any static macvlan IP, verify it's genuinely free — `ping` alone is not sufficient, since a device that's simply not currently responding can still "own" the IP via ARP:
```bash
ping -c 2 <candidate-ip>
arp -a | grep <candidate-ip>
```
Both must return empty. Also check the IP falls **outside** your router's DHCP-assignable range to prevent future conflicts.

---

## 5. Initial Setup (Preflight)

1. Start the stack:
```bash
cd /volume1/<username>/docker/graylog/
sudo docker compose up -d
```
2. Retrieve the temporary Preflight password:
```bash
sudo docker logs graylog-graylog-1 2>&1 | grep -A 2 "Initial configuration"
```
3. From a separate device on the LAN, browse to `http://<macvlan-ip>:9000/` and log in with `admin` + the temporary password.
4. **Create new CA** → set certificate lifetime (365 days used in this deployment) → **Create CA**.
5. **Provision certificates** for the Data Node → wait for success.
6. **Resume startup**.
7. Log into the real Graylog UI with `admin` + the plaintext password behind `GRAYLOG_ROOT_PASSWORD_SHA2`.

---

## 6. Syslog Input Configuration

1. **System → Inputs** → select **Syslog UDP** → **Launch new input**.
2. Settings:
   - Title: `Network Switches`
   - Bind address: `0.0.0.0`
   - Port: `<PORT>`
3. Save and confirm status shows **Running**.
4. When prompted, choose **Route to a new Stream** → name it `Network Switches`.

> **Port note:** Port `514` (the syslog standard) generally cannot be bound directly by Graylog inside the container, since Graylog does not run as root and Linux restricts binding to ports below 1024.

---

## 7. Switch-Side Configuration

### 7.1 Cisco IOS (Catalyst-family switches)

```
enable
configure terminal
logging host <macvlan ip> transport udp port 5140
logging trap informational
logging on
service timestamps log datetime localtime
end
write memory
```

**`service timestamps log datetime localtime`** is required — by default, Cisco IOS timestamps syslog messages in UTC internally, even when `show clock` correctly displays local time. Without this, all timestamps in Graylog will be offset (in this deployment, 8 hours behind actual local time).

### 7.2 Ensure a default gateway is set (Layer 2 switches)

Switches without `ip routing` enabled rely on a single default gateway. If missing, the switch cannot reach Graylog (or anything outside its own VLAN) at all:
```
show ip default-gateway
```
If it shows `0.0.0.0`:
```
configure terminal
ip default-gateway <core router IP>
end
write memory
```
Find the correct gateway via CDP if unknown:
```
show cdp neighbors detail
```

### 7.3 Consistent source IP (optional but recommended)

By default, a switch's outbound syslog packets use whichever interface routing naturally selects which may not match the IP used to manage the switch. To force a consistent, predictable source IP:
```
configure terminal
logging source-interface <interface-name>
end
write memory
```

### 7.4 Login event logging

**Standard IOS platforms:**
```
configure terminal
login on-failure log
login on-success log
end
write memory
```

**Cisco SG350/SG300 (Small Business series) — different platform, different syntax:**
```
configure terminal
aaa logging login
end
write memory
```



