---
title: Network VPN route
draft: true
tags:
  - network
date: 2024-01-01
---
# Network VPN route


```powershell
add-VpnConnectionRoute -ConnectionName "My VPN" -DestinationPrefix "192.168.31.0/24"

ping 192.168.31.2

$conn = get-vpnconnection -ConnectionName "My VPN"
$conn.routes

Remove-VpnConnectionRoute -ConnectionName "My VPN" -DestinationPrefix "192.168.31.0/24"

add-VpnConnectionRoute -ConnectionName "My VPN" -DestinationPrefix "192.168.31.0/27"

# -AllUserConnection
```
