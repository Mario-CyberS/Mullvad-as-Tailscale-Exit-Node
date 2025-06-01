# Mullvad as Tailscale Exit Node  
This project documents how to configure a **Raspberry Pi** to act as a secure internet gateway for a connected PC using **Mullvad VPN** (via WireGuard) while still allowing **remote access over Tailscale**. It ensures **all PC traffic is routed through Mullvad**, while preserving RDP and SSH access from a MacBook using Tailscale.

---

## 🎯 Objective  
To route a Windows PC's entire internet traffic through Mullvad VPN via a Raspberry Pi while maintaining remote RDP access through Tailscale. This enables strong privacy and filtered DNS while retaining secure admin control.

---

## 🔍 Why Use Mullvad with Tailscale?  
This setup combines the privacy of Mullvad with the mesh access power of Tailscale:
- Mullvad anonymizes outbound traffic through WireGuard VPN  
- Tailscale maintains RDP/SSH access over a private mesh network  
- No need for public IPs or router port forwarding  
- Strong DNS and firewall rules prevent leaks  

---

## 📚 Skills Learned  
- WireGuard VPN configuration and routing  
- Split-DNS with dnsmasq and Tailscale DNS fallback  
- Advanced iptables firewall and NAT chaining  
- SSH/RDP access control via Tailscale ACLs and tags  
- Managing persistent Linux services and forwarding  

---

## 🛠️ Tools Used  
<div>
  <img src="https://img.shields.io/badge/-Raspberry%20Pi-C51A4A?style=for-the-badge&logo=Raspberry-Pi&logoColor=white"/>
  <img src="https://img.shields.io/badge/-WireGuard-88171A?style=for-the-badge&logo=WireGuard&logoColor=white"/>
  <img src="https://img.shields.io/badge/-Tailscale-005AE0?style=for-the-badge&logo=Tailscale&logoColor=white"/>
  <img src="https://img.shields.io/badge/-dnsmasq-lightgrey?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/-iptables-DD3333?style=for-the-badge"/>
</div>

---

## 📝 Deployment Steps  

### 1. Install Dependencies on Raspberry Pi  
```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install wireguard resolvconf netfilter-persistent dnsmasq -y
```

### 2. Get Mullvad WireGuard Config
- Go to https://mullvad.net/account
- Under WireGuard Configuration, choose:
-- Platform: Linux
-- Location: e.g., USA > Raleigh
-- Server: pick 1 server
-- Protocol: IPv4
-- Tunnel: Only IPv4
-- Port: 51820
-- Use IP addresses: ✅ Yes
-- Content Blocking: Optional
- Click Generate
- Download the zip file then extract```bash













