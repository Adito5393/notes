---
title: Windows VPN route
draft: false
tags:
  - network
date: 2024-06-18
description: Add a route to a Windows VPN connection.
---
Get [all available VPN connections](https://learn.microsoft.com/en-us/powershell/module/vpnclient/get-vpnconnection):
```powershell
Get-VpnConnection
```

If it's not there, then get all available VPN connections from the global phonebook:
```powershell
Get-VpnConnection -AllUserConnection
```

To add a route: (use `-AllUserConnection` at the end as needed):
```powershell
add-VpnConnectionRoute -ConnectionName "My VPN" -DestinationPrefix "192.168.31.0/24"
```

To verify it:
```powershell
$conn = get-vpnconnection -ConnectionName "My VPN"
$conn.routes
```

If you need to remove a route:
```powershell
Remove-VpnConnectionRoute -ConnectionName "My VPN" -DestinationPrefix "192.168.31.0/24"
```