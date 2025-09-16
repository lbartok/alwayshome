# Always Home VPN Installation Guide

This guide provides step-by-step instructions for setting up a complete "always at home" VPN solution using ZeroTier One as the VPN transport. The setup includes:

1. **Raspberry Pi** with ZeroTier One and IP forwarding
2. **Teltonika Router** connected to ZeroTier network
3. **Network connectivity** via LAN and WLAN

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Raspberry Pi Setup](#raspberry-pi-setup)
3. [ZeroTier One Installation on Raspberry Pi](#zerotier-one-installation-on-raspberry-pi)
4. [IP Forwarding Configuration](#ip-forwarding-configuration)
5. [Teltonika Router Setup](#teltonika-router-setup)
6. [ZeroTier Network Configuration](#zerotier-network-configuration)
7. [Network Connectivity Setup](#network-connectivity-setup)
8. [Testing and Verification](#testing-and-verification)
9. [Troubleshooting](#troubleshooting)

## Prerequisites

### Hardware Requirements
- Raspberry Pi (Model 3B+ or newer recommended)
- MicroSD card (16GB or larger)
- Teltonika router (RUT models supported)
- Network cables
- Power supplies for both devices

### Software Requirements
- Raspberry Pi OS (latest version)
- ZeroTier account (free tier available) (https://my.zerotier.com/)
- SSH client for remote configuration

### Network Requirements
- Internet connection for initial setup
- Access to router configuration interface

## Raspberry Pi Setup

### 1. Install Raspberry Pi OS

1. Download the latest Raspberry Pi OS from [official website](https://www.raspberrypi.org/software/)
2. Flash the image to your MicroSD card using Raspberry Pi Imager
3. Enable SSH by creating an empty file named `ssh` in the boot partition
4. Configure WiFi (optional) by creating `wpa_supplicant.conf` in boot partition:

```conf
country=US
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="Your_WiFi_Name"
    psk="Your_WiFi_Password"
}
```

### 2. Initial Configuration

1. Boot the Raspberry Pi and connect via SSH:
```bash
ssh pi@raspberrypi.local
```

2. Update the system:
```bash
sudo apt update && sudo apt upgrade -y
```

3. Configure the Pi using raspi-config:
```bash
sudo raspi-config
```
- Enable SSH permanently
- Set timezone
- Expand filesystem
- Configure memory split if needed

## ZeroTier One Installation on Raspberry Pi

### 1. Install ZeroTier One

```bash
# Download and install ZeroTier One
curl -s https://install.zerotier.com | sudo bash

# Verify installation
sudo zerotier-cli info
```

### 2. Join ZeroTier Network

1. Create a ZeroTier network at [my.zerotier.com](https://my.zerotier.com)
2. Note your Network ID (16-character string)
3. Join the network:

```bash
sudo zerotier-cli join YOUR_NETWORK_ID
```

4. Authorize the device in ZeroTier Central:
   - Go to your network in ZeroTier Central
   - Find your Raspberry Pi in the members list
   - Check the "Auth?" checkbox to authorize

### 3. Configure ZeroTier Interface

1. Check ZeroTier interface status:
```bash
sudo zerotier-cli listnetworks
ip addr show zt0
```

2. Note the assigned ZeroTier IP address (typically in 10.x.x.x range)

## IP Forwarding Configuration

### 1. Enable IP Forwarding

1. Edit sysctl configuration:
```bash
sudo nano /etc/sysctl.conf
```

2. Uncomment or add the following line:
```conf
net.ipv4.ip_forward=1
```

3. Apply changes immediately:
```bash
sudo sysctl -p
```

### 2. Configure iptables for NAT

1. Create NAT rules for traffic forwarding:
```bash
# Enable NAT for ZeroTier traffic to local network
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/8 -o eth0 -j MASQUERADE
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/8 -o wlan0 -j MASQUERADE

# Allow forwarding between interfaces
sudo iptables -A FORWARD -i zt+ -o eth0 -j ACCEPT
sudo iptables -A FORWARD -i zt+ -o wlan0 -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o zt+ -j ACCEPT
sudo iptables -A FORWARD -i wlan0 -o zt+ -j ACCEPT
```

2. Save iptables rules:
```bash
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```

3. Make rules persistent by editing `/etc/rc.local`:
```bash
sudo nano /etc/rc.local
```

Add before `exit 0`:
```bash
iptables-restore < /etc/iptables.ipv4.nat
```

### 3. Configure Route Advertisement

1. Create a startup script for route management:
```bash
sudo nano /etc/systemd/system/zerotier-routes.service
```

2. Add the following content:
```ini
[Unit]
Description=ZeroTier Route Management
After=zerotier-one.service
Requires=zerotier-one.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash -c 'sleep 10 && zerotier-cli set YOUR_NETWORK_ID allowDefault=1'

[Install]
WantedBy=multi-user.target
```

3. Enable the service:
```bash
sudo systemctl enable zerotier-routes.service
```

## Teltonika Router Setup

### 1. Initial Router Configuration

1. Connect to router's web interface (usually `192.168.1.1`)
2. Complete initial setup wizard
3. Update firmware to latest version
4. Configure WAN connection (cellular/ethernet)

### 2. Install ZeroTier Package

#### For RUT routers with Package Manager:

1. Navigate to **System → Package Manager**
2. Search for "zerotier"
3. Install the ZeroTier package
4. Reboot the router

#### For Manual Installation:

1. Download ZeroTier package for OpenWrt:
```bash
# SSH into router
ssh root@192.168.1.1

# Download and install ZeroTier
opkg update
opkg install zerotier
```

### 3. Configure ZeroTier on Router

1. Via Web Interface (if available):
   - Navigate to **Services → ZeroTier**
   - Enable ZeroTier service
   - Enter your Network ID
   - Save configuration

2. Via Command Line:
```bash
# SSH into router
ssh root@192.168.1.1

# Start ZeroTier service
/etc/init.d/zerotier enable
/etc/init.d/zerotier start

# Join ZeroTier network
zerotier-cli join YOUR_NETWORK_ID

# Verify connection
zerotier-cli listnetworks
```

3. Authorize the router in ZeroTier Central (same as Raspberry Pi)

### 4. Configure Router Networking

1. Edit network configuration:
```bash
# Edit network config
vi /etc/config/network
```

2. Add ZeroTier interface:
```uci
config interface 'zerotier'
    option ifname 'ztbvhyxyxy'  # Replace with actual ZT interface
    option proto 'none'
```

3. Configure firewall rules:
```bash
# Edit firewall config
vi /etc/config/firewall
```

Add zone for ZeroTier:
```uci
config zone
    option name 'zerotier'
    option input 'ACCEPT'
    option output 'ACCEPT'
    option forward 'ACCEPT'
    option network 'zerotier'

config forwarding
    option src 'lan'
    option dest 'zerotier'

config forwarding
    option src 'zerotier'
    option dest 'lan'
```

## ZeroTier Network Configuration

### 1. Network Settings in ZeroTier Central

1. Go to [my.zerotier.com](https://my.zerotier.com)
2. Select your network
3. Configure the following settings:

#### IPv4 Auto-Assign
- **Auto-Assign from Range**: Enable
- **Range**: `10.147.20.0/24` (or your preferred range)

#### Advanced Settings
- **Allow Ethernet Bridging**: Enable
- **Allow Default Route Override**: Enable

### 2. Route Configuration

Add routes for local networks:

1. In ZeroTier Central, go to **Routes** section
2. Add routes for each location:
   - Raspberry Pi location: `192.168.1.0/24` via `Pi_ZT_IP`
   - Router location: `192.168.2.0/24` via `Router_ZT_IP`

### 3. Member Authorization

Ensure all devices are authorized:
- ✅ Raspberry Pi
- ✅ Teltonika Router
- ✅ Client devices (phones, laptops, etc.)

## Network Connectivity Setup

### 1. LAN Connectivity

#### Raspberry Pi LAN Setup:
```bash
# Configure static IP for local network
sudo nano /etc/dhcpcd.conf

# Add at the end:
interface eth0
static ip_address=192.168.1.100/24
static routers=192.168.1.1
static domain_name_servers=8.8.8.8 8.8.4.4
```

#### Router LAN Configuration:
1. Set LAN IP range: `192.168.2.1/24`
2. Enable DHCP: `192.168.2.100-192.168.2.200`
3. Configure DNS servers

### 2. WLAN Connectivity

#### Raspberry Pi WLAN Setup:
```bash
# Configure WiFi connection
sudo nano /etc/wpa_supplicant/wpa_supplicant.conf

# Add network block:
network={
    ssid="Your_Network"
    psk="Your_Password"
    priority=1
}
```

#### Router WLAN Configuration:
1. Navigate to **Network → Wireless**
2. Configure SSID and password
3. Set security to WPA2-PSK
4. Enable wireless network

### 3. Traffic Routing Configuration

Create routing rules for seamless connectivity:

#### On Raspberry Pi:
```bash
# Add route for router's local network
sudo ip route add 192.168.2.0/24 via ROUTER_ZT_IP dev zt0

# Make persistent
echo "ip route add 192.168.2.0/24 via ROUTER_ZT_IP dev zt0" | sudo tee -a /etc/rc.local
```

#### On Teltonika Router:
```bash
# Add route for Pi's local network
route add -net 192.168.1.0/24 gw PI_ZT_IP dev ztxxxxxxxx
```

## Testing and Verification

### 1. Connectivity Tests

#### Basic ZeroTier Connectivity:
```bash
# From any ZeroTier device, ping other members
ping PI_ZT_IP
ping ROUTER_ZT_IP

# Check ZeroTier status
zerotier-cli listnetworks
zerotier-cli peers
```

#### Cross-Network Connectivity:
```bash
# From device connected to Pi's network, test router network
ping 192.168.2.1

# From device connected to router's network, test Pi network
ping 192.168.1.100
```

### 2. Performance Testing

```bash
# Test bandwidth between networks
iperf3 -s  # On one device
iperf3 -c TARGET_IP  # On another device

# Test latency
ping -c 10 TARGET_IP
```

### 3. Service Status Verification

#### Raspberry Pi:
```bash
# Check ZeroTier service
sudo systemctl status zerotier-one

# Check IP forwarding
cat /proc/sys/net/ipv4/ip_forward

# Check iptables rules
sudo iptables -L -n -v
```

#### Teltonika Router:
```bash
# Check ZeroTier service
/etc/init.d/zerotier status

# Check network interfaces
ip addr show
```

## Troubleshooting

### Common Issues and Solutions

#### 1. ZeroTier Connection Issues

**Problem**: Device shows as offline in ZeroTier Central
```bash
# Solution: Restart ZeroTier service
sudo systemctl restart zerotier-one

# Check firewall settings
sudo ufw status
```

**Problem**: Cannot join network
```bash
# Verify network ID is correct
zerotier-cli info
zerotier-cli listnetworks

# Leave and rejoin network
sudo zerotier-cli leave NETWORK_ID
sudo zerotier-cli join NETWORK_ID
```

#### 2. Routing Issues

**Problem**: Cannot reach devices on remote network
```bash
# Check routing table
ip route show

# Verify IP forwarding is enabled
cat /proc/sys/net/ipv4/ip_forward

# Check iptables rules
sudo iptables -L FORWARD -v
```

#### 3. Performance Issues

**Problem**: Slow connection speeds
- Check ZeroTier planet servers: `zerotier-cli peers`
- Verify direct connection: Look for "DIRECT" in peers list
- Consider custom root servers for better performance

#### 4. Router-Specific Issues

**Problem**: ZeroTier package not available
```bash
# Update package lists
opkg update

# Install from alternative source
opkg install http://downloads.openwrt.org/releases/packages/arm_cortex-a9/packages/zerotier_*.ipk
```

### Log Files and Debugging

#### Raspberry Pi Logs:
```bash
# ZeroTier logs
sudo journalctl -u zerotier-one -f

# System logs
sudo tail -f /var/log/syslog
```

#### Router Logs:
```bash
# System logs
logread -f

# ZeroTier specific logs
logread | grep zerotier
```

### Network Diagnostics

```bash
# Trace route to verify path
traceroute TARGET_IP

# Check DNS resolution
nslookup google.com

# Monitor network traffic
sudo tcpdump -i zt0 icmp
```

## Security Considerations

1. **Change default passwords** on all devices
2. **Enable firewall rules** to restrict unnecessary access
3. **Regularly update** all software components
4. **Monitor ZeroTier network** for unauthorized devices
5. **Use strong encryption** for WiFi networks
6. **Limit administrative access** to management interfaces

## Maintenance

### Regular Tasks

1. **Monthly**: Update system packages
2. **Quarterly**: Review ZeroTier network members
3. **Annually**: Rotate passwords and certificates

### Backup Configuration

```bash
# Backup ZeroTier identity
sudo cp -r /var/lib/zerotier-one/ /backup/zerotier-backup/

# Backup router configuration
scp root@router:/etc/config/* /backup/router-config/
```

---

This installation guide provides a complete setup for an "always at home" VPN solution using ZeroTier One transport with Raspberry Pi and Teltonika router integration. For additional support, consult the [ZeroTier documentation](https://docs.zerotier.com/) and [Teltonika wiki](https://wiki.teltonika-networks.com/).
