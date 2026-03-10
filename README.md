# 🖥️ Raspberry Pi RDP — Remote Access Outside Your Local Network

> Access your Raspberry Pi's graphical desktop from anywhere on the internet using RDP and Cloudflare Tunnel.

---

## 📖 Overview

This guide documents how to set up **Remote Desktop Protocol (RDP)** access to a Raspberry Pi and expose it securely to the internet using **Cloudflare Tunnel** — without opening any ports on your router or exposing your home IP address.

This project builds on a previous setup where SSH access to the Pi was made publicly available via a Cloudflare Tunnel and a custom domain. You can find that guide here:
👉 [cloudflare-ssh-connection](https://github.com/MarvelousCyberKnight/cloudlfaresshconnection)

---

## 🧰 Prerequisites

Before following this guide, make sure you have:

- A Raspberry Pi with a desktop environment installed (e.g., Raspberry Pi OS with Desktop)
- Cloudflare account with a domain already configured
- **Docker** and **Docker Compose** installed on the Raspberry Pi
- `cloudflared` installed on your **client machine**
- SSH access already working (as set up in the linked project above)

---

## 🔧 Setup Steps

### 1. Install an RDP Server on the Raspberry Pi

Raspberry Pi OS does not ship with an RDP server by default. Install `xrdp`, which acts as an RDP-compatible server:

```bash
sudo apt update
sudo apt install xrdp -y
sudo systemctl enable xrdp
sudo systemctl start xrdp
```

Verify the service is running:

```bash
sudo systemctl status xrdp
```

> `xrdp` listens on **port 3389** by default.

---

### 2. Test RDP Over Your Local Network

Before exposing the connection externally, verify that RDP works within your LAN.

On your **client machine** (Windows, macOS, or Linux):

- **Windows**: Open **Remote Desktop Connection** and enter your Pi's local IP (e.g., `192.168.1.x`)
- **macOS/Linux**: Use a client such as [Microsoft Remote Desktop](https://apps.apple.com/app/microsoft-remote-desktop/id1295203466) or `remmina`

Log in with your Pi's username and password. If the desktop loads, local RDP is working correctly.

---

### 3. Create a Cloudflare Tunnel for RDP Using Docker Compose

Instead of installing `cloudflared` directly on the Pi, the tunnel is run as a **Docker container** using Docker Compose. This keeps the setup clean, portable, and easy to manage.

#### 3a. Get Your Cloudflare Tunnel Token

1. Go to the [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/)
2. Navigate to **Networks → Tunnels → Create a Tunnel**
3. Choose **Cloudflared** as the connector type
4. Name your tunnel (e.g., `rdp-tunnel`) and save
5. Under **Public Hostnames**, add:
   - **Subdomain**: `rdp`
   - **Domain**: `yourdomain.com`
   - **Service Type**: `RDP`
   - **URL**: `localhost:3389`
6. Copy the **tunnel token** shown in the install step — you'll need it below

#### 3b. Create the Docker Compose File

On your **Raspberry Pi**, create a `docker-compose.yml` file:

```yaml
# docker-compose.yml

services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared-rdp
    restart: unless-stopped
    network_mode: host
    command: tunnel --no-autoupdate run --token <YOUR_TUNNEL_TOKEN>
```

> Replace `<YOUR_TUNNEL_TOKEN>` with the token copied from the Cloudflare dashboard.
>
> `network_mode: host` is important — it allows the container to reach `localhost:3389` (your Pi's xrdp service) directly.

#### 3c. Start the Tunnel

```bash
docker compose up -d
```

Verify the container is running:

```bash
docker compose logs -f
```

You should see a message confirming the tunnel is connected to Cloudflare's network. The tunnel will automatically restart on reboot thanks to `restart: unless-stopped`.

---

### 4. Connect from Your Client Machine

Because the RDP traffic is tunnelled through Cloudflare, you cannot connect directly with a standard RDP client. Instead, you use `cloudflared` on the **client side** to create a local TCP proxy.

On your **client machine**, run:

```bash
cloudflared access tcp --hostname rdp.yourdomain.com --url localhost:3389
```

This command:
- Connects to your Cloudflare Tunnel
- Forwards the remote RDP port to `localhost:3389` on your local machine

Now open your RDP client and connect to:

```
localhost:3389
```

You will be prompted to log in with your Raspberry Pi credentials, and the full desktop will load — accessed securely from anywhere in the world.

---

## 🏗️ Architecture

```
Client Machine
──────────────
[RDP Client] ──► localhost:3389
                      │
              [cloudflared proxy]
                      │
                      ▼
              Cloudflare Network
                      │
                      ▼
           [Cloudflare Tunnel (cloudflared)]
                      │
              Raspberry Pi (Server)
              ──────────────────────
              localhost:3389 ──► [xrdp] ──► Desktop Session
```

---

## 🔐 Security Notes

- All traffic is **encrypted end-to-end** through the Cloudflare Tunnel — no raw RDP is exposed to the internet.
- No ports need to be opened on your **home router or firewall**.
- Your **home IP address is never exposed**.
- You can add **Cloudflare Access policies** (e.g., email OTP, Google login) to the tunnel hostname for an additional authentication layer.

---

## 📚 Related Projects

| Project | Description |
|---|---|
| [cloudflare-ssh-connection](https://github.com/MarvelousCyberKnight/cloudlfaresshconnection) | SSH access to Raspberry Pi via Cloudflare Tunnel |
| This project | RDP (graphical desktop) access via Cloudflare Tunnel |

---

## 📄 License

This project is open for personal and educational use. Feel free to fork and adapt it for your own homelab setup.
