# 39 — Windows Server Administration

> **[← Index](00_INDEX.md)** | **Related: [Windows CLI](04_Windows_CLI.md) · [Active Directory](09_Active_Directory.md) · [IIS](10_IIS.md) · [PowerShell Scripting](31_PowerShell_Scripting.md) · [Security Concepts](14_Security_Concepts.md)**

---

## Windows Server Editions

| Edition | Target | Key Features |
|---------|--------|-------------|
| **Standard** | Small/medium orgs | Up to 2 VMs with Hyper-V |
| **Datacenter** | Large enterprises | Unlimited VMs, all features |
| **Essentials** | ≤ 25 users / 50 devices | Simplified management |
| **Core** | Server without GUI | Lower footprint, more secure |
| **Nano Server** | Containers, cloud | Minimal footprint |

---

## Server Manager & Remote Administration

```powershell
# ── Install Server Manager tools ──────────────────────
# RSAT (Remote Server Administration Tools) — manage from workstation
Add-WindowsCapability -Online -Name "Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0"
Add-WindowsCapability -Online -Name "Rsat.DHCP.Tools~~~~0.0.1.0"
Add-WindowsCapability -Online -Name "Rsat.DNS.Tools~~~~0.0.1.0"
Add-WindowsCapability -Online -Name "Rsat.ServerManager.Tools~~~~0.0.1.0"

# ── Enable WinRM (Windows Remote Management) ──────────
# On target server:
Enable-PSRemoting -Force
winrm quickconfig

# ── Enable RDP ────────────────────────────────────────
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' `
    -Name fDenyTSConnections -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
```

---

## Roles & Features

```powershell
# List all available roles and features
Get-WindowsFeature | Where-Object Installed | Format-Table Name, DisplayName

# Install common roles
Install-WindowsFeature -Name Web-Server -IncludeManagementTools          # IIS
Install-WindowsFeature -Name DNS -IncludeManagementTools                  # DNS Server
Install-WindowsFeature -Name DHCP -IncludeManagementTools                 # DHCP Server
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools   # AD DS
Install-WindowsFeature -Name FileAndStorage-Services                      # File Server
Install-WindowsFeature -Name Hyper-V -IncludeManagementTools             # Hyper-V
Install-WindowsFeature -Name NPAS -IncludeSubFeatures                    # NPS (RADIUS)
Install-WindowsFeature -Name Print-Server -IncludeManagementTools        # Print Server

# Remove a role
Uninstall-WindowsFeature -Name Web-Server -IncludeManagementTools

# Promote server to Domain Controller
Import-Module ADDSDeployment
Install-ADDSForest `
    -DomainName "corp.example.com" `
    -DomainNetbiosName "CORP" `
    -DomainMode "WinThreshold" `
    -ForestMode "WinThreshold" `
    -InstallDns `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd!" -AsPlainText -Force) `
    -Force
```

---

## DHCP Server Management

```powershell
Import-Module DhcpServer

# ── Authorize DHCP server in AD ───────────────────────
Add-DhcpServerInDC -DnsName "dhcp01.corp.example.com" -IPAddress 10.0.0.10

# ── Create scope (subnet) ─────────────────────────────
Add-DhcpServerv4Scope `
    -Name "Office LAN" `
    -StartRange 10.0.0.100 `
    -EndRange   10.0.0.200 `
    -SubnetMask 255.255.255.0 `
    -Description "Office network" `
    -LeaseDuration (New-TimeSpan -Days 8) `
    -State Active

# ── Scope options (DNS, gateway) ──────────────────────
Set-DhcpServerv4OptionValue `
    -ScopeId 10.0.0.0 `
    -DnsServer 10.0.0.1, 8.8.8.8 `
    -DnsDomain "corp.example.com" `
    -Router 10.0.0.1

# ── Exclusions (IPs not to assign) ────────────────────
Add-DhcpServerv4ExclusionRange `
    -ScopeId 10.0.0.0 `
    -StartRange 10.0.0.100 `
    -EndRange 10.0.0.110     # Reserved for static devices

# ── Reservations (always same IP for a device) ────────
Add-DhcpServerv4Reservation `
    -ScopeId 10.0.0.0 `
    -IPAddress 10.0.0.50 `
    -ClientId "AA-BB-CC-DD-EE-FF" `    # MAC address
    -Name "Printer-01" `
    -Description "Office printer"

# ── View leases ───────────────────────────────────────
Get-DhcpServerv4Lease -ScopeId 10.0.0.0
Get-DhcpServerv4Lease -ScopeId 10.0.0.0 | Where-Object AddressState -eq Active
Get-DhcpServerv4Lease -ScopeId 10.0.0.0 | Export-Csv leases.csv

# ── DHCP failover (high availability) ─────────────────
Add-DhcpServerv4Failover `
    -Name "DHCP-Failover" `
    -PartnerServer "dhcp02.corp.example.com" `
    -ScopeId 10.0.0.0 `
    -Mode LoadBalance `
    -LoadBalancePercent 50 `
    -SharedSecret "SharedSecret123!"
```

---

## Windows DNS Server

```powershell
Import-Module DnsServer

# ── Zones ─────────────────────────────────────────────
Get-DnsServerZone
Add-DnsServerPrimaryZone -Name "example.com" -ReplicationScope "Domain" -DynamicUpdate Secure
Add-DnsServerSecondaryZone -Name "example.com" -ZoneFile "example.com.dns" `
    -MasterServers 10.0.0.1

# ── Records ───────────────────────────────────────────
Add-DnsServerResourceRecordA    -ZoneName "example.com" -Name "web"   -IPv4Address "10.0.0.20"
Add-DnsServerResourceRecordAAAA -ZoneName "example.com" -Name "web"   -IPv6Address "2001:db8::1"
Add-DnsServerResourceRecordCName -ZoneName "example.com" -Name "www"  -HostNameAlias "web.example.com."
Add-DnsServerResourceRecordMx   -ZoneName "example.com" -Name "@"     -MailExchange "mail.example.com." -Preference 10
Add-DnsServerResourceRecord -Txt -ZoneName "example.com" -Name "@"    -DescriptiveText "v=spf1 ip4:10.0.0.0/24 ~all"

# ── Reverse zone ──────────────────────────────────────
Add-DnsServerPrimaryZone -NetworkId "10.0.0.0/24" -ReplicationScope "Domain"
Add-DnsServerResourceRecordPtr -ZoneName "0.0.10.in-addr.arpa" -Name "20" `
    -PtrDomainName "web.example.com."

# ── View / Remove ─────────────────────────────────────
Get-DnsServerResourceRecord -ZoneName "example.com"
Get-DnsServerResourceRecord -ZoneName "example.com" -Name "web" -RRType A
Remove-DnsServerResourceRecord -ZoneName "example.com" -Name "web" -RRType A -Force

# ── Cache / flush ─────────────────────────────────────
Get-DnsServerCache
Clear-DnsServerCache -Force
```

---

## Windows File Server

```powershell
# ── Create SMB share ──────────────────────────────────
# Create directory
New-Item -Path "D:\Shares\Projects" -ItemType Directory

# Create share
New-SmbShare `
    -Name "Projects" `
    -Path "D:\Shares\Projects" `
    -Description "Project files" `
    -FullAccess "CORP\IT-Admins" `
    -ChangeAccess "CORP\Developers" `
    -ReadAccess "CORP\Domain Users" `
    -FolderEnumerationMode AccessBased   # Hide folders user can't access

# View shares
Get-SmbShare
Get-SmbShare -Name "Projects"

# View active sessions / open files
Get-SmbSession
Get-SmbOpenFile

# Close a session
Close-SmbSession -SessionId 1234 -Force
Close-SmbOpenFile -FileId 5678 -Force

# Remove share
Remove-SmbShare -Name "Projects" -Force

# ── NTFS permissions ──────────────────────────────────
$path = "D:\Shares\Projects"
$acl  = Get-Acl $path

# Add permission
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "CORP\Developers", "Modify", "ContainerInherit,ObjectInherit", "None", "Allow"
)
$acl.AddAccessRule($rule)
Set-Acl $path $acl

# Disable inheritance and copy existing
$acl.SetAccessRuleProtection($true, $true)   # Block inherit, copy existing
Set-Acl $path $acl

# View permissions
(Get-Acl $path).Access | Format-Table IdentityReference, FileSystemRights, AccessControlType

# ── Disk quota ────────────────────────────────────────
# Enable quota on volume
Enable-FileSystemQuota -Path D:\ -DefaultLimit 10GB -DefaultWarningLimit 9GB

# ── Shadow copies (VSS) ───────────────────────────────
# Enable Volume Shadow Copy
vssadmin list shadows                   # List snapshots
vssadmin create shadow /for=D:          # Create snapshot
# Configure via GUI: right-click drive → Configure Shadow Copies
```

---

## Windows Update Management

```powershell
# ── Check Windows Update ──────────────────────────────
# PSWindowsUpdate module
Install-Module PSWindowsUpdate -Force
Import-Module PSWindowsUpdate

Get-WUList                              # List available updates
Install-WindowsUpdate -AcceptAll -AutoReboot    # Install all + reboot
Install-WindowsUpdate -KBArticleID "KB5001234"  # Specific update
Get-WUHistory                           # Update history

# ── WSUS — Windows Server Update Services ─────────────
# Client configuration via Group Policy or registry
$wsusServer = "http://wsus.corp.example.com:8530"
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" `
    -Name WUServer -Value $wsusServer
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" `
    -Name WUStatusServer -Value $wsusServer
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" `
    -Name UseWUServer -Value 1

# Force check-in to WSUS
wuauclt /resetauthorization /detectnow
```

---

## Windows Performance Monitoring

```powershell
# ── Resource Monitor snapshots ────────────────────────
# CPU
Get-Counter '\Processor(_Total)\% Processor Time' -SampleInterval 2 -MaxSamples 5

# Memory
Get-Counter '\Memory\Available MBytes'
Get-Counter '\Memory\Pages/sec'

# Disk
Get-Counter '\PhysicalDisk(_Total)\Disk Reads/sec'
Get-Counter '\PhysicalDisk(_Total)\Disk Writes/sec'
Get-Counter '\LogicalDisk(C:)\% Free Space'

# Network
Get-Counter '\Network Interface(*)\Bytes Total/sec'

# ── Data Collector Sets (long-term capture) ───────────
# Create via Performance Monitor GUI: perfmon.exe
# Or PowerShell:
$dcs = New-Object -COM Pla.DataCollectorSet
$dcs.DisplayName = "Server Performance"
# ... configure and schedule

# ── Reliability Monitor ───────────────────────────────
# View in GUI: perfmon /rel
# Or PowerShell:
Get-WinEvent -LogName "Microsoft-Windows-Reliability*" -MaxEvents 20

# ── Process deep-dive ─────────────────────────────────
Get-Process | Sort-Object WorkingSet64 -Descending | Select-Object -First 10 `
    Name, Id, CPU,
    @{N='RAM_MB'; E={[math]::Round($_.WorkingSet64/1MB,1)}},
    @{N='Handles'; E={$_.HandleCount}}
```

---

## Windows Registry

```powershell
# Navigate registry as drive
cd HKLM:\SOFTWARE\Microsoft
Get-ChildItem HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run

# Read value
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion"
(Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion").ProductName

# Create key
New-Item -Path "HKCU:\SOFTWARE\MyApp" -Force

# Set value
Set-ItemProperty -Path "HKCU:\SOFTWARE\MyApp" -Name "Setting1" -Value "Hello"
Set-ItemProperty -Path "HKCU:\SOFTWARE\MyApp" -Name "Count" -Value 42 -Type DWord

# Delete value
Remove-ItemProperty -Path "HKCU:\SOFTWARE\MyApp" -Name "Setting1"

# Delete key
Remove-Item -Path "HKCU:\SOFTWARE\MyApp" -Recurse

# Backup registry key
reg export "HKCU\SOFTWARE\MyApp" "C:\backup\myapp-reg.reg"
reg import "C:\backup\myapp-reg.reg"

# Common registry paths
HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion          # OS info
HKLM:\SYSTEM\CurrentControlSet\Services\                    # Services
HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run         # Auto-start programs
HKLM:\SOFTWARE\Policies\Microsoft\Windows\                  # Group Policy results
HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\   # User shell settings
```

---

## Windows Event Forwarding (WEF)

```powershell
# Centralize logs from multiple servers
# On collector server:
wecutil qc                    # Configure Windows Event Collector service
wecutil cs subscription.xml   # Create subscription

# subscription.xml example
<Subscription>
  <SubscriptionId>Security-Events</SubscriptionId>
  <SubscriptionType>SourceInitiated</SubscriptionType>
  <Description>Collect security events</Description>
  <Enabled>true</Enabled>
  <Uri>http://schemas.microsoft.com/wbem/wsman/1/windows/EventLog</Uri>
  <ConfigurationMode>MinLatency</ConfigurationMode>
  <Query>
    <![CDATA[<QueryList><Query Path="Security">
      <Select>*[System[(EventID=4624 or EventID=4625 or EventID=4740)]]</Select>
    </Query></QueryList>]]>
  </Query>
  <AllowedSourceDomainComputers>O:NSG:NSD:(A;;GA;;;DC)</AllowedSourceDomainComputers>
</Subscription>

# On source servers:
winrm quickconfig
# GPO: Computer → Administrative Templates → Windows Components →
#      Event Forwarding → Configure target subscription manager
#      Value: Server=http://collector.corp.example.com:5985/wsman/SubscriptionManager/WEC
```

---

## Useful Admin One-Liners

```powershell
# Find large files on C: drive
Get-ChildItem C:\ -Recurse -ErrorAction SilentlyContinue |
    Sort-Object Length -Descending | Select-Object -First 20 FullName,
    @{N='Size_MB';E={[math]::Round($_.Length/1MB,2)}}

# Services set to auto that are not running
Get-Service | Where-Object {$_.StartType -eq 'Automatic' -and $_.Status -ne 'Running'} |
    Select-Object Name, Status, DisplayName

# Last 10 system errors
Get-EventLog System -EntryType Error -Newest 10 | Format-Table TimeGenerated, Source, Message -Wrap

# Open network ports with process names
Get-NetTCPConnection -State Listen |
    Select-Object LocalPort, @{N='Process';E={(Get-Process -Id $_.OwningProcess).Name}} |
    Sort-Object LocalPort

# Check disk health
Get-PhysicalDisk | Select-Object FriendlyName, OperationalStatus, HealthStatus, Size

# Pending reboots check
$reboots = @{
    'CBS'      = Test-Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\RebootPending"
    'WU'       = Test-Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update\RebootRequired"
    'PendRen'  = (Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager" -EA SilentlyContinue).PendingFileRenameOperations
}
$reboots | Format-List
```

---

## Related Topics

- [Windows CLI ←](04_Windows_CLI.md)
- [Active Directory ←](09_Active_Directory.md)
- [IIS ←](10_IIS.md)
- [PowerShell Scripting ←](31_PowerShell_Scripting.md)
- [Security Concepts ←](14_Security_Concepts.md)
- [Monitoring & Logging ←](13_Monitoring_Logging.md)
- [NTP ←](11_NTP.md) — W32TM on Windows

---

> [Index](00_INDEX.md)
