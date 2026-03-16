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
Go to https://mullvad.net/account
Under WireGuard Configuration, choose:
 - Platform: Linux
 - Location: e.g., USA > Raleigh
 - Server: pick 1 server
 - Protocol: IPv4
 - Tunnel: Only IPv4
 - Port: 51820
 - Use IP addresses: ✅ Yes
 - Content Blocking: Optional

Click Generate

Download the zip file then extract

### 3. Transfer Config to Pi
From your Mac:
```bash
scp ~/Downloads/your-config.conf owner@raspbpi.local:~ 
```
Then on Pi:
```bash
sudo mv ~/your-config.conf /etc/wireguard/mullvad.conf 
sudo chmod 600 /etc/wireguard/mullvad.conf 
```
Edit config line to route traffic without breaking Tailscale:
```bash
sudo nano /etc/wireguard/mullvad.conf
```
```bash
AllowedIPs = 0.0.0.0/1, 128.0.0.0/1
```
And change DNS to:
```bash
DNS = 1.1.1.1
```

### 4. Split-DNS on Pi with dnsmasq
Step 1: Confirm dnsmasq Is Installed
```bash
sudo apt update
sudo apt install dnsmasq -y
```
Step 2: Configure /etc/dnsmasq.conf
Edit:
```bash
sudo nano /etc/dnsmasq.conf
```
At the bottom, add:
```bash
# ===== LAN service on eth1 =====
interface=eth1
bind-interfaces
listen-address=127.0.0.1,192.168.50.1

# ===== DHCP for Windows PC / LAN =====
dhcp-range=192.168.50.10,192.168.50.100,12h
dhcp-option=option:router,192.168.50.1
dhcp-option=option:dns-server,192.168.50.1
dhcp-host=**:**:**:**:**:**,192.168.50.10

# ===== DNS behavior =====
no-resolv
domain-needed
bogus-priv

# Tailscale MagicDNS / tailnet names
server=/ts.net/100.100.100.100

# Everything else through Mullvad resolver
server=10.64.0.1

# ===== Logging (optional, useful for debugging) =====
log-queries
log-dhcp

# ===== Cache =====
cache-size=1000
```
Explanation:
PC gets DHCP and DNS from the Pi on eth1
DNS requests to .ts.net go to Tailscale DNS
Everything else goes to Mullvad’s internal resolver or Cloudflare
no-resolv tells dnsmasq to ignore /etc/resolv.conf
listen-address=127.0.0.1 ensures the Pi queries dnsmasq locally

Step 3: Set up resolv.conf
Unlock resolv.conf:
```bash
sudo chattr -i /etc/resolv.conf
```
Edit:
```bash
sudo nano /etc/resolv.conf
```
Set:
```bash
nameserver 100.100.100.100
nameserver 8.8.8.8
nameserver 1.1.1.1
```
Then lock it:
```bash
sudo chattr +i /etc/resolv.conf
```
Step 4: Restart dnsmasq and Verify
```bash
sudo systemctl restart dnsmasq
sudo systemctl status dnsmasq
```
You should see it running.

Make config back-up:
```bash
sudo cp /etc/dnsmasq.conf /etc/dnsmasq.conf.bak
```
Test DNS:
```bash
dig google.com @127.0.0.1
dig tailnet-name.ts.net @127.0.0.1
```
Step 5: Block All Other DNS Attempts from PC
This prevents DNS leaks:
```bash
# Allow PC to use Pi for DNS
sudo iptables -A FORWARD -i eth1 -p udp --dport 53 -j ACCEPT
sudo iptables -A FORWARD -i eth1 -p tcp --dport 53 -j ACCEPT
# Drop any other DNS (from PC to internet)
sudo iptables -A FORWARD -p udp --dport 53 -j DROP
sudo iptables -A FORWARD -p tcp --dport 53 -j DROP
# Save
sudo netfilter-persistent save
```

### 5. Enable IP Forwarding
```bash
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward 
sudo nano /etc/sysctl.conf
```
Ensure this line is added or uncommented:
```bash
net.ipv4.ip_forward=1
```
Then apply:
```bash
sudo sysctl -p 
```

### 6. Apply iptables Rules (Before VPN Up)
```bash
sudo nano /usr/local/sbin/apply-gateway-firewall.sh
```
```bash
#!/bin/bash
set -e

LAN_IF="eth1"
VPN_IF="mullvad"
TS_IF="tailscale0"

PC_IP="192.168.50.10"
PI_LAN_IP="192.168.50.1"
LAN_SUBNET="192.168.50.0/24"
TS_CGNAT="100.64.0.0/10"

# Flush old rules
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X
iptables -t mangle -F
iptables -t mangle -X

# Default policies
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# INPUT

iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p icmp -j ACCEPT
iptables -A INPUT -p udp --dport 41641 -j ACCEPT
iptables -A INPUT -i "$TS_IF" -j ACCEPT

# DNS to Pi from LAN
iptables -A INPUT -i "$LAN_IF" -s "$LAN_SUBNET" -d "$PI_LAN_IP" -p udp --dport 53 -j ACCEPT
iptables -A INPUT -i "$LAN_IF" -s "$LAN_SUBNET" -d "$PI_LAN_IP" -p tcp --dport 53 -j ACCEPT

# DHCP to Pi
iptables -A INPUT -i "$LAN_IF" -p udp --sport 68 --dport 67 -j ACCEPT

# Optional OpenSSH on Pi only over Tailscale
iptables -A INPUT -i "$TS_IF" -p tcp --dport 22 -j ACCEPT

# FORWARD

iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Only allow LAN subnet from eth1
iptables -A FORWARD -i "$LAN_IF" ! -s "$LAN_SUBNET" -j DROP

# Block direct DNS leaks from LAN clients to internet
iptables -A FORWARD -i "$LAN_IF" -s "$LAN_SUBNET" -p udp --dport 53 -j REJECT
iptables -A FORWARD -i "$LAN_IF" -s "$LAN_SUBNET" -p tcp --dport 53 -j REJECT

# Allow LAN web traffic out via Mullvad
iptables -A FORWARD -i "$LAN_IF" -s "$LAN_SUBNET" -o "$VPN_IF" -p tcp --dport 443 -j ACCEPT
iptables -A FORWARD -i "$LAN_IF" -s "$LAN_SUBNET" -o "$VPN_IF" -p tcp --dport 80 -j ACCEPT
iptables -A FORWARD -i "$LAN_IF" -s "$LAN_SUBNET" -o "$VPN_IF" -p udp --dport 123 -j ACCEPT

# Allow Tailscale to reach the PC for subnet-routed RDP if needed
iptables -A FORWARD -i "$TS_IF" -o "$LAN_IF" -d "$PC_IP" -p tcp --dport 3389 -j ACCEPT
iptables -A FORWARD -i "$TS_IF" -o "$LAN_IF" -d "$PC_IP" -p icmp -j ACCEPT

# Log/drop remaining LAN forwards
iptables -A FORWARD -i "$LAN_IF" -s "$LAN_SUBNET" -m limit --limit 5/min --limit-burst 10 -j LOG --log-prefix "FW_DROP_LAN: " --log-level 4
iptables -A FORWARD -i "$LAN_IF" -s "$LAN_SUBNET" -j DROP

iptables -A FORWARD -m limit --limit 5/min --limit-burst 10 -j LOG --log-prefix "FW_DROP_OTHER: " --log-level 4
iptables -A FORWARD -j DROP

# NAT

iptables -t nat -A POSTROUTING -s "$LAN_SUBNET" -o "$VPN_IF" -j MASQUERADE
iptables -t nat -A POSTROUTING -s "$TS_CGNAT" -o "$VPN_IF" -j MASQUERADE
```
```bash
sudo /usr/local/sbin/apply-gateway-firewall.sh 
sudo netfilter-persistent save
```
Inspect current rules:
```bash
sudo iptables -S
sudo iptables -t nat -S
```

### 7. Start VPN (WireGuard)
```bash
sudo wg-quick up mullvad
```
```bash
sudo wg show
```
(Optional: Auto-enable on boot)
```bash
sudo systemctl enable wg-quick@mullvad 
```

### 8. Confirm It’s Working
On Pi:
```bash
curl https://api.ipify.org 
curl https://am.i.mullvad.net/json 
ping google.com
ping 1.1.1.1
```
On PC:
```bash
Invoke-RestMethod https://api.ipify.org 
```
Check ipconfig to confirm gateway is 192.168.50.1
On PC open browser: visit Am I Mullvad? 
You should now see your Mullvad VPN IP instead of your router or ISP.

### 9. Upgrade ssh into Pi from Mac
First on the Tailscale ACLs Dashboard enable Tailscale ssh. 
- You will see the button to enable next to the edit ACL section
Then we want to update our ACLs on the Tailscale dashboard:
```bash
{
  "acls": [
    {
      "action": "accept",
      "src": ["Mario-CyberS@github"],
      "dst": ["tag:home:3389"]
    },
    {
      "action": "accept",
      "src": ["Mario-CyberS@github"],
      "dst": ["tag:relay-node:22"]
    }
  ],

  "tagOwners": {
    "tag:home": ["Mario-CyberS@github"],
    "tag:relay-node": ["Mario-CyberS@github"]
  },

  "ssh": [
    {
      "action": "accept",
      "src": ["Mario-CyberS@github"],
      "dst": ["tag:relay-node"],
      "users": ["owner"]
    },
    {
      "action": "check",
      "src": ["Mario-CyberS@github"],
      "dst": ["tag:relay-node"],
      "users": ["root"],
      "checkPeriod": "12h"
    }
  ],

  "tests": [
    {
      "src": "Mario-CyberS@github",
      "proto": "tcp",
      "accept": ["tag:home:3389", "tag:relay-node:22"],
      "deny": [
        "tag:home:22",
        "tag:relay-node:80",
        "tag:relay-node:443",
        "tag:relay-node:3389"
      ]
    }
  ],

  "sshTests": [
    {
      "src": "Mario-CyberS@github",
      "dst": ["tag:relay-node"],
      "accept": ["owner"],
      "check": ["root"],
      "deny": ["pi", "ubuntu", "admin"]
    }
  ]
}
```
Next on your Pi run this:
```bash
sudo tailscale up --ssh
```
Run this on Pi:
```bash
sudo systemctl restart tailscaled
```
Test ssh from mac:
```bash
ssh owner@<Pi-Tailscale-IP>
```

### Caution: If RDP from Mac to PC breaks Fix Tailscale DNS + RDP
Run on Pi:
```bash
sudo tailscale up --reset
sudo tailscale up --ssh --accept-dns=true --accept-routes=true --advertise-tags=tag:relay-node
sudo systemctl restart tailscaled
```
On Mac:
```bash
tailscale up --accept-dns=true --reset
```
Then test:
```bash
nslookup homelab.tail85e488.ts.net
```
→ Should return 100.x.x.x of your PC.


This setup now:
- Preserves your original Pi → PC NAT/firewall
- Lets your Mac RDP into the PC via Tailscale
- Routes all PC outbound traffic through Mullvad VPN
- Keeps DNS locked and avoids Tailscale overwrites

---

### 👨‍💻 Author
Mario Tagaras | Cyber Security Specialist

