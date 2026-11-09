# SSH and Network Connectivity Troubleshooting

## Project Overview

This project documents troubleshooting a Raspberry Pi 5 that lost SSH connectivity.

The Raspberry Pi was connected to a TP-Link Omada ES208G managed switch. My Windows 11 PC provided internet access through Wi-Fi using Internet Connection Sharing (ICS).

The main issues encountered were:
- SSH connection timeouts.
- Ethernet DHCP address assignment failures.
- The Raspberry Pi having no usable IPv4 address.
- Internet connectivity failures despite a working physical Ethernet link.

## Network Configuration

| Device | Interface | Address |
|---|---|---|
| Windows PC | Wi-Fi | `192.168.1.70` |
| Windows PC | Ethernet | `192.168.137.1` |
| Raspberry Pi 5 | eth0 | `192.168.137.250` |
| Default Gateway | Windows Ethernet | `192.168.137.1` |
| DNS server | Raspberry Pi | `1.1.1.1` |

The Raspberry Pi temporarily used a static IP address because DHCP through Windows ICS was failing.

## 1. Initial SSH Troubleshooting

From Windows PowerShell, I attempted to connect using the hostname:

```powershell
ssh labadmin@pi-lab.local
```

The connection timed out.

I also used:

```powershell
ipconfig
```

To inspect Windows network adapter addresses, and:

```powershell
arp -a
```

To inspect the ARP table and identify devices visible on the local network.

## 2. Raspberry Pi Network Diagnostics

Using the Raspberry Pi's local console, I inspected the Ethernet interface and network management status.

### Commands used

Check network interface status:

```bash
ip link show eth0
```

Check IPv4 address assignment:

```bash
ip -4 addr show eth0
```

Display all network devices:

```bash
nmcli device status
```

Check saved network connections:

```bash
nmcli connection show
```

Check NetworkManager service status:

```bash
sudo systemctl status NetworkManager --no-pager
```

Inspect recent NetworkManager logs:

```bash
sudo journalctl -u NetworkManager -n 30 --no-pager
```

Check the current routing table:

```bash
ip route
```

Display assigned IP addresses:

```bash
hostname -I
```

Attempt to connect Ethernet through NetworkManager:

```bash
sudo nmcli device connect eth0
```

Restart NetworkManager's networking:

```bash
sudo nmcli networking off
sudo nmcli networking on
```

The Ethernet interface reported a physical link (`LOWER_UP`), but DHCP was failing with an IP configuration error.

## 3. Windows Internet Connection Sharing Diagnostics

Windows was configured to share its Wi-Fi internet connection through its Ethernet adapter.

I inspected Windows services with:

```powershell
Get-Service SharedAccess, Dhcp
```

Both the Internet Connection Sharing service and DHCP Client service were running.

I also inspected network configuration using:

```powershell
Get-NetIPConfiguration
```

And checked the Ethernet IPv4 protocol binding:

```powershell
Get-NetAdapterBinding -Name "Ethernet" -ComponentID ms_tcpip
```

To inspect the ICS service configuration:

```powershell
Get-CimInstance Win32_Service -Filter "Name='SharedAccess'" | Select-Object Name, State, StartMode, PathName
```

Restarted Internet Connection Sharing:

```powershell
Restart-Service -Name SharedAccess -Force
```

Tested Windows internet connectivity:

```powershell
ping 8.8.8.8
```

## 4. Inspecting Windows Internet Connection Sharing

I used PowerShell's Windows networking COM interface to inspect which adapters had Internet Connection Sharing enabled.

```powershell
$share = New-Object -ComObject HNetCfg.HNetShare

$share.EnumEveryConnection() | ForEach-Object {
    $props = $share.NetConnectionProps($_)
    $cfg = $share.INetSharingConfigurationForINetConnection($_)

    [PSCustomObject]@{
        Name = $props.Name
        Device = $props.DeviceName
        SharingEnabled = $cfg.SharingEnabled
        SharingType = if ($cfg.SharingEnabled) {
            $cfg.SharingConnectionType
        } else {
            "-"
        }
    }
} | Format-Table -AutoSize
```

The inspection showed that Internet Connection Sharing was also enabled on an extra VirtualBox Host-Only Ethernet adapter named `Ethernet 2`.

I disabled sharing on that extra adapter:

```powershell
$share = New-Object -ComObject HNetCfg.HNetShare

$conn = $share.EnumEveryConnection() | Where-Object {
    $share.NetConnectionProps($_).Name -eq "Ethernet 2"
}

$share.INetSharingConfigurationForINetConnection($conn).DisableSharing()
```

After disabling sharing on the extra adapter, the Raspberry Pi successfully regained external IP connectivity.

## 5. Configuring a Temporary Static IP Address

Because DHCP was failing, I temporarily assigned a static IP address to the Raspberry Pi using NetworkManager.

First, I tested the proposed address:

```bash
ping -c 3 192.168.137.250
```

Then created a static Ethernet connection:

```bash
sudo nmcli connection add type ethernet ifname eth0 con-name pi-static ipv4.method manual ipv4.addresses 192.168.137.250/24 ipv4.gateway 192.168.137.1 ipv4.dns 1.1.1.1
```

Activated the connection:

```bash
sudo nmcli connection up pi-static
```

This provided the Raspberry Pi with a usable address and allowed SSH access to be restored.

## 6. Connectivity Testing

### Test Raspberry Pi to Windows

```bash
ping -c 4 192.168.137.1
```

### Test Windows to Raspberry Pi

```powershell
ping 192.168.137.250
```

### Test external IP connectivity

```bash
ping -c 4 8.8.8.8
```

### Test DNS resolution

```bash
ping -c 4 google.com
```

Additional DNS test:

```bash
ping -c 4 mlb.com
```

### Test HTTPS connectivity

```bash
curl -4 -I --max-time 10 https://www.google.com
```

```bash
curl -4 -I --max-time 10 https://www.mlb.com
```

These tests helped distinguish local network connectivity, external IP connectivity, DNS resolution, and actual HTTPS access.

## 7. Restoring SSH Access

Once the static IP connection was active, I connected from Windows PowerShell using:

```powershell
ssh labadmin@192.168.137.250
```

This restored remote access without relying on hostname resolution.

## Final Results

- Restored SSH access to the Raspberry Pi.
- Verified the physical Ethernet link.
- Identified DHCP address assignment failures.
- Investigated Windows Internet Connection Sharing.
- Identified an additional ICS configuration on the VirtualBox Host-Only adapter.
- Disabled sharing on the extra adapter.
- Configured a temporary static IP address.
- Restored internet access.
- Tested connectivity between Windows and Raspberry Pi.
- Verified external IP, DNS, and HTTPS connectivity.

## Skills Demonstrated

- Linux network troubleshooting
- Windows network adapter administration
- DHCP troubleshooting
- Static IPv4 configuration
- NetworkManager
- Internet Connection Sharing
- DNS troubleshooting
- SSH remote administration
- PowerShell administration
- Network connectivity testing
- Technical troubleshooting and documentation

## Lessons Learned

This troubleshooting experience demonstrated the importance of checking multiple layers of network connectivity.

A physical Ethernet link does not guarantee that a device has an IP address, a default gateway, or internet access.

Using tools such as `ip`, `nmcli`, `journalctl`, `ping`, and PowerShell helped isolate the problems and restore connectivity.

The static IP configuration was a temporary workaround. DHCP should be investigated and restored as the long-term configuration before removing the temporary profile.

