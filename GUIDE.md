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
* AdGuard Home using `br0` network on Unraid **or** running on a standalone device such as a raspberry pi.
* Tailscale installed on:
  * Unraid (or device running Adguard)
  * Client devices
* A domain hosted on a public DNS resolver (example: cloudflare)

---

## 🚀 Step 1: Prepare Unraid WebGUI

Migrate the Unraid WebGUI ports to something other than 80 and 443

### Unraid WebGUI ➜ Settings ➜ Management Access

| Port Type | Original Port  | New Port |
| --------- | -------------- | -------- |
| HTTP      | 80             | 81       |
| HTTPS     | 443            | 4430     |

<img width="1110" height="781" alt="image" src="https://github.com/user-attachments/assets/38bd9cf3-d69c-4584-992b-a74f726b9dd0" />


> [!IMPORTANT]
>  When you migrate the ports, you will need to access unraid using the port you specified otherwise you will fail to connect.
> 
>  Example: `http://10.0.0.2:81` or `https://10.0.0.2:4430`

---

## 🔀 Step 2: Install and Configure NGINX Reverse Proxy

1. Install **NGINX Proxy Manager** of **SWAG** from Community Applications
2. Set **NGINX Proxy Manager** to `host` networking mode

In **NGINX Proxy Manager**:

### Create a Proxy Host

<img width="500" height="546" alt="image" src="https://github.com/user-attachments/assets/c02de96e-5405-4928-b431-4adebe831aa1" />


> [!NOTE]
> Containers or Services running in `bridge` network mode will use the unraid server's local IP.

Repeat adding a proxy host entry for each container:

```
radarr.example.com → 10.0.0.2:7878
  plex.example.com → 10.0.0.2:32400
```

---

## 📦 Step 3: Install and Configure AdGuard Home

1. Install **AdGuard Home**
2. Change Container Network to `br0` and assign a static IP 
<img width="650" height="559" alt="image" src="https://github.com/user-attachments/assets/4373a066-19a4-4b7b-a35b-6d071c5b343a" />

> [!IMPORTANT]
>  For Unraid to communicate with conatiners using `br0` networking, you must enable "**Host access to custom networks**" within the Unraid Docker settings.
>
> <img width="823" height="818" alt="image" src="https://github.com/user-attachments/assets/d842f353-0acc-442e-bd91-d25ace1644b2" />

---

## 🌍 Step 4: Configure Wildcard DNS Rewrite

### AdGuard UI ➜ Filters ➜ DNS Rewrites

Add a wildcard rule:

<img width="505" height="555" alt="image" src="https://github.com/user-attachments/assets/55b1681c-91de-438a-b8ae-5ffbf235be4d" />


>[!TIP]
>The wildcard rules will allow you to access all subdomains, pinging them should return the IP of NGINX/UNRAID
>
>`abcd.example.com → 10.0.0.2`
>
>`wxyz.example.com → 10.0.0.2`

---


## 🧠 Step 5: Point LAN Clients to AdGuard

Update router or DHCP:

* Set DNS server to AdGuard IP or manually configure on devices:

>[!NOTE]
>You will need to use the IP addresses you've assigned, not the ones shown in this guide

Example: 
DNS Server: `10.0.0.3`

Verify: `ping app.example.com` should return IP of NGINX/UNRAID `10.0.0.2`

---

## 🛡️ Step 6: Install & Configure Tailscale

>[!NOTE]
> This setup assumes **AdGuard is the authoritative DNS server** for both LAN and Tailscale traffic.

### Install Tailscale

* Install Tailscale on:

  * Unraid (or device running AdGuard)
  * All client devices

>[!IMPORTANT]
>**AdGuard & NGINX must be on the same Tailscale network**

### Install Tailscale Unraid Plugin

1. Apps → Plugins → Tailscale
<img width="578" height="462" alt="image" src="https://github.com/user-attachments/assets/cd5fcac5-3ffd-4a87-8230-cfdd80ccba32" />


2. Settings → Taiscale
3. Add Unraid to your tailnet and authenticate device.
4. Once Unraid shows up in your tailnet return to the Tailscale Plugin settings page
5. At the bottom of the page where it says **Advertised Routes** add your LAN subnet/24

Example: `192.168.0.0/24 or 172.16.0.0/12 or 10.0.0.0/8` depending on your specific subnet.
<img width="489" height="107" alt="image" src="https://github.com/user-attachments/assets/f0217d7f-399a-4cd1-ac30-1d41037e1641" />


>[!WARNING]
>
>Advertising the entire subnet range will allow access all of the LAN devices as if you were on your local network. Any other user who you may invite to your tailnet will have the same access.

>[!TIP]
>
>You can instead opt to only advertise specific IP addressess such as using the IP/32 CIDR notation
>
>Example: `10.0.0.2/32 and 10.0.0.3/32` you will need to advertise each IP address you need accessible from tailscale.

### In Tailscale Admin Console

1. Click on **Unraid** and approve the subnet route.
<img width="809" height="474" alt="image" src="https://github.com/user-attachments/assets/c8c3201b-0327-488d-b8c5-cfda32a4db0a" />


2. Go to **DNS → Nameservers**
<img width="493" height="404" alt="image" src="https://github.com/user-attachments/assets/7ca48fe7-c3d6-4da1-aa17-db86ccd80c56" />


3. Enable **Override local DNS** and add the **Local static IP** you assigned to Adguard (or device running AdGuard)
<img width="778" height="844" alt="image" src="https://github.com/user-attachments/assets/da9e24d3-7a07-4322-af4f-3df911eaf7b2" />



>[!TIP]
>
>Alternatively you if you installed Adguard on a standalone device, you can instead set the tailscale dns override to the IP tailscale has assigned to the device.
>
><img width="696" height="86" alt="image" src="https://github.com/user-attachments/assets/f04e7974-4f30-461d-85d1-6f4060029b71" />


All Tailscale devices now resolve DNS via AdGuard
