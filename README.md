# QNX Target Network Setup Guide for T264 Thor Silicon

This guide provides detailed steps to set up networking on a QNX target device after flashing the QNX image from a host machine. This documentation is based on the T264 Silicon Bringup Information and practical implementation.

## Overview

After flashing a QNX image on a Thor target from a host machine, you need to establish network connectivity between the host and target for development and testing purposes. This guide covers both local network setup and external network access configuration.

## Prerequisites

- QNX image successfully flashed on Thor target (T264)
- Host machine with Ubuntu/Linux
- Physical connection between host and target via Ethernet
- Administrator/sudo access on both host and target

## Network Interface Information

### For T264 Targets:
- **Target Interface**: `mgbe3_0` (Multi-Gigabit Ethernet interface)
- **Host Interface**: Typically `eno1`, `enp39s0`, or `enp44s0f0` (varies by system)

### IP Address Scheme:
- **Host IP**: `192.168.56.200`
- **Target IP**: `192.168.56.201` or `192.168.56.50`
- **Netmask**: `255.255.255.0`

## Step-by-Step Network Setup

### 1. Host Machine Configuration

#### Check Existing Network Connections
```bash
nmcli connection show
```

#### Setup Tegra Ethernet Network (if not exists)
```bash
# Add ethernet connection
sudo nmcli con add type ethernet con-name tegra-ethernet-network ifname enp39s0 ip4 192.168.56.200

# Bring up the connection
sudo nmcli connection up tegra-ethernet-network

# Verify connection status
sudo nmcli connection show
```

#### Alternative Interface Setup (Manual)
```bash
# Bring down interface first
sudo ifconfig enp39s0 down

# Configure interface with IP and netmask
sudo ifconfig enp39s0 192.168.56.200 netmask 255.255.255.0 up
```

#### Cleanup Conflicting Interfaces
```bash
# Check for other active interfaces
nmcli connection show

# Bring down any conflicting interfaces (except enp39s0, enp44s0f0, or your main network interface)
sudo ifconfig <device-name> down
```

### 2. Target Device Configuration

#### Basic Network Interface Setup
```bash
# For T264 targets - configure MGBE3_0 interface
ifconfig mgbe3_0 192.168.56.201 up

# Alternative IP configuration
sudo ifconfig mgbe3_0 192.168.56.50 up
```

#### Verify Network Interface
```bash
# Check interface status
ifconfig mgbe3_0

# Check all interfaces
ifconfig
```

### 3. Test Network Connectivity

#### From Host to Target
```bash
# Ping target from host
ping 192.168.56.201

# Or if using alternative IP
ping 192.168.56.50
```

#### From Target to Host
```bash
# Ping host from target
ping 192.168.56.200
```

## Advanced Configuration

### NFS Setup (Optional)

#### On Host Machine
```bash
# Edit exports file
sudo vim /etc/exports

# Add NFS export line
/data2 192.168.56.201(async,rw,no_root_squash,no_all_squash,no_subtree_check)

# Apply export changes
sudo exportfs -a
```

#### On Target Device
```bash
# Mount NFS share
fs-nfs3 192.168.56.200:/data2 /root
```

### External Network Access

If you need the target to access external networks (internet):

#### On Host Machine
```bash
# Disable firewall
sudo ufw disable

# Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1

# Configure iptables for NAT (replace ens1f0 with your internet-facing interface)
sudo iptables -t nat -A POSTROUTING -o ens1f0 -j MASQUERADE
sudo iptables -A FORWARD -i ens1f0 -j ACCEPT
sudo iptables -A FORWARD -o enp39s0 -j ACCEPT
```

#### On Target Device
```bash
# Add default route through host
sudo route add default gw 192.168.56.200 mgbe3_0

# Configure DNS (Method 1 - systemd-resolved)
sudo sed -i '/#DNS/a DNS=10.61.13.53' /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved

# Configure DNS (Method 2 - direct resolv.conf)
echo "nameserver 10.61.13.53" | sudo tee /etc/resolv.conf

# Test external connectivity
ping google.com
```

## Troubleshooting

### Common Issues and Solutions

#### 1. Interface Not Found
```bash
# List all available interfaces
ifconfig -a

# Check if interface name is different (e.g., eqos_0 instead of mgbe3_0)
ifconfig eqos_0 192.168.56.201 up
```

#### 2. Permission Denied
```bash
# Ensure you have root/sudo privileges
sudo su -

# Then run the ifconfig commands
```

#### 3. Network Interface Already Configured
```bash
# Reset interface
sudo ifconfig mgbe3_0 down
sudo ifconfig mgbe3_0 192.168.56.201 up
```

#### 4. DNS Resolution Issues
```bash
# Check current DNS servers on host
resolvectl status | grep "DNS Server" -A4

# Use the DNS server from your host system
# Example: Current DNS Server: 10.63.120.27
echo "nameserver 10.63.120.27" | sudo tee /etc/resolv.conf
```

### Network Interface Variations by Platform

| Platform | Target Interface | Notes |
|----------|------------------|-------|
| T264 SLT | mgbe3_0 | Standard for T264 |
| Ferrix | mgbe3_0 | Multi-Gigabit Ethernet |
| General | eqos_0 | Fallback interface |

### Port and Interface Mapping

- **Host Ports**: Usually `enp39s0` or `enp44s0f0`
- **Target Ports**: `mgbe3_0` for T264 Thor silicon
- **Alternative**: Some setups may use `eqos_0`

## Summary Commands

### Quick Setup Script (Host)
```bash
#!/bin/bash
# Quick host setup for QNX target networking

# Configure host interface
sudo nmcli con add type ethernet con-name tegra-ethernet-network ifname enp39s0 ip4 192.168.56.200
sudo nmcli connection up tegra-ethernet-network

# Alternative manual setup
# sudo ifconfig enp39s0 down
# sudo ifconfig enp39s0 192.168.56.200 netmask 255.255.255.0 up

echo "Host configured. Target IP should be 192.168.56.201"
echo "Test with: ping 192.168.56.201"
```

### Quick Setup Script (Target)
```bash
#!/bin/bash
# Quick target setup for QNX networking

# Configure target interface
ifconfig mgbe3_0 192.168.56.201 up

# Test connectivity
ping -c 3 192.168.56.200

echo "Target configured. Test from host with: ping 192.168.56.201"
```

## References

- T264 Silicon Bringup Information: https://confluence.nvidia.com/display/DEVTOOLS/T264+Silicon+Bringup+Information
- Thor Ethernet Silicon Bringup: https://confluence.nvidia.com/display/CONNECTIVITYSW/Thor+Ethernet+Silicon+Bringup
- T264 SW Bringup: https://confluence.nvidia.com/display/TCS/T264+SW+Bringup

---

**Last Updated**: September 2025  
**Platform**: T264 Thor Silicon with QNX
