# 🌐 NGINX Reverse Proxy on Unraid with AdGuard DNS & Tailscale

> This guide assumes basic familiarity with Unraid, Docker, and networking concepts. The guide also assumes you provide your own domain and dns services.

This guide walks through setting up an **NGINX reverse proxy on Unraid** to access local Docker containers using **friendly web URLs** instead of IP:PORT combinations.

In this guide we will:

* Resolve domains via **AdGuard Home** using a **wildcard DNS rewrite** rule
* Forward requests from **AdGuard Home** to **NGINX Proxy Manager** 
* Override DNS in **Tailscale** so everything works seamlessly on and off your LAN

---

## 🧩 Requirements

* Unraid server with Docker enabled
* NGINX Proxy Manager **or** SWAG
* AdGuard Home using `br0` network on Unraid **or** running on a standalone device
* Tailscale installed on:
  * Unraid
  * (Optional) Device running AdGuard Home
  * Client devices
* A domain hosted on a public DNS resolver (example: cloudflare)

---

## 🚀 Step 1: Prepare Unraid WebGUI

Migrate the Unraid WebGUI ports to something other than 80 and 443

### Unraid WebGUI ➜ Settings ➜ Network Management

| Port Type | Original Port  | New Port |
| --------- | -------------- | -------- |
| HTTP      | 80             | 81       |
| HTTPS     | 443            | 4430     |

> [!IMPORTANT]
>  When you migrate the ports, you will need to access unraid using the port you specified otherwise you will fail to connect.
> 
>  Example: `http://192.168.1.2:81` or `https://192.168.1.2:4430`

---

## 🔀 Step 2: Install and Configure NGINX Reverse Proxy

1. Install **NGINX Proxy Manager** of **SWAG** from Community Applications
2. Set **NGINX Proxy Manager** to `host` networking mode

In **NGINX Proxy Manager**:

### Create a Proxy Host

| Field            | Value             |
| ---------------- | ----------------- |
| Domain Name      | `app.example.com` |
| Forward Hostname | `UNRAID_IP`       |
| Forward Port     | `PORT`            |
| Scheme           | `http`            |

> [!NOTE]
> Containers or Services running in `bridge` network mode will use the unraid server's local IP where `UNRAID_IP` is shown
> 
> Example: `192.168.1.2`

Enable:

* Websockets (if needed)
* Block Common Exploits
* HTTPS/HTST

Repeat for each container:

```
sonarr.example.com → 192.168.1.2:8989
radarr.example.com → 192.168.1.2:7878
  plex.example.com → 192.168.1.2:32400
```

---

## 📦 Step 3: Install and Configure AdGuard Home

1. Install **AdGuard Home** (Unraid container or Standalone device)
2. Change Container Network to `br0` and assign a static IP 
3. Expose ports:

| Port      | Purpose     |
| --------- | ----------- |
| 53        | DNS         |
| 3000      | Setup UI    |
| Static IP | 192.168.1.3 |

> [!IMPORTANT]
>  For Unraid to communicate with conatiners using `br0` networking, you must enable "**Host access to custom networks**" within the Unraid Docker settings.
>
> Unraid Settings → Docker → Host access to custom network: yes

---

## 🌍 Step 4: Configure Wildcard DNS Rewrite

### AdGuard UI ➜ Filters ➜ DNS Rewrites

Add a wildcard rule:

Example:

```
*.example.com → 192.168.1.2
```

>[!TIP]
>The wildcard rules will allow you to access all subdomains, pinging them should return the IP of NGINX/UNRAID
>
>`abcd.example.com → 192.168.1.2`
>
>`wxyz.example.com → 192.168.1.2`

---


## 🧠 Step 5: Point LAN Clients to AdGuard

Update router or DHCP:

* Set DNS server to AdGuard IP or manually configure on devices:

>[!NOTE]
>You will need to use the IP addresses you've assigned, not the ones shown in this guide

Example: 
DNS Server: `192.168.1.3`

Verify: `ping app.example.com` should return IP of NGINX/UNRAID `192.168.1.2`

---

## 🛡️ Step 6: Install & Configure Tailscale

>[!NOTE]
> This setup assumes **AdGuard is the authoritative DNS server** for both LAN and Tailscale traffic.

### Install Tailscale

* Install Tailscale on:

  * Unraid and Adguard Device if standalone
  * All client devices

>[!IMPORTANT]
>**AdGuard & NGINX must be on the same Tailscale network**

### In Tailscale Admin Console

1. Go to **DNS → Nameservers**
2. Enable **Override local DNS**
3. Add the **Tailscale IP or Local IP of Unraid or AdGuard**

Example:

```
100.64.0.10 or 192.168.1.3
```

All Tailscale devices now resolve DNS via AdGuard
