# Troubleshooting: chrome_win_cis

Errors hit while preparing or testing the Windows 11 lab VM.

## T-W1: Windows VM has no network card in `ipconfig`

```
(ipconfig shows no Ethernet adapter, although the Proxmox VM has network devices attached)
```

- Cause: Proxmox gives new VMs a VirtIO network card by default; Windows has no VirtIO driver built in (Device
  Manager shows "Ethernet Controller" under Other devices).
- Fix: Proxmox, Hardware, Network Device, Model = Intel E1000E, then shut down and start the VM. Or install
  `virtio-win-guest-tools.exe` from the virtio-win ISO.
- Prevent: pick E1000E when creating Windows VMs, or attach the virtio-win ISO at install time.

## T-W2: control node can't ping Windows, Windows can ping the control node

```
$ ping -c2 192.168.20.60
PING 192.168.20.60 (192.168.20.60) 56(84) bytes of data.
2 packets transmitted, 0 received, 100% packet loss, time 1050ms
```

- Cause: Windows Firewall blocks inbound ICMP echo by default; outbound pings work because replies to traffic
  Windows started are allowed back in. The lab card (no gateway) is also marked Public.
- Fix (PowerShell as Administrator):
  ```powershell
  New-NetFirewallRule -DisplayName "Allow ICMPv4 ping" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow
  Get-NetConnectionProfile | Set-NetConnectionProfile -NetworkCategory Private
  ```
- Prevent: part of the lab prep in `build-guide.md` section 2.1.

## T-W3: WinRM port 5985 times out from the control node

```
$ nc -zv 192.168.20.60 5985
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: TIMEOUT.
```

- Cause: a timeout (not "refused") means the firewall drops the packets: the WinRM firewall rule was missing or did
  not cover the card's profile (WinRM itself was set up before the card was switched to Private).
- Fix: `Enable-PSRemoting -Force` again after setting the cards to Private, or add the rule:
  ```powershell
  New-NetFirewallRule -DisplayName "WinRM HTTP 5985" -Protocol TCP -LocalPort 5985 -Direction Inbound -Action Allow -Profile Any
  ```
  Then `nc -zv 192.168.20.60 5985` shows `Connected`.
- Prevent: set the network profile to Private before running `Enable-PSRemoting`.
