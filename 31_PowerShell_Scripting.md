# 31 — PowerShell Scripting

> **[← Index](00_INDEX.md)** | **Related: [Windows CLI](04_Windows_CLI.md) · [Services & Processes](15_Services_Processes.md) · [Active Directory](09_Active_Directory.md) · [CI/CD](27_CICD_Fundamentals.md)**

---

## Why PowerShell Scripting?

PowerShell is far more than a shell — it is a full scripting language built on **.NET**, with:
- **Object pipeline** — pass rich objects, not text
- **Cmdlets** — consistent `Verb-Noun` naming
- **Modules** — reusable libraries (AD, Exchange, Azure, AWS)
- **Remote execution** — run scripts on hundreds of servers
- **Cross-platform** — PowerShell 7+ runs on Linux and macOS

---

## Script Structure and Best Practices

```powershell
#Requires -Version 7.0
#Requires -Modules ActiveDirectory

<#
.SYNOPSIS
    Creates AD user accounts from a CSV file.

.DESCRIPTION
    Reads a CSV of new employees and creates AD accounts,
    sets initial password, and emails credentials.

.PARAMETER CsvPath
    Path to the input CSV file.

.PARAMETER Domain
    Target AD domain. Defaults to current domain.

.EXAMPLE
    .\New-BulkADUsers.ps1 -CsvPath ".\newusers.csv"

.NOTES
    Author:  Alice Smith
    Version: 1.0
    Date:    2024-04-22
#>

[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory, HelpMessage = "Path to CSV file")]
    [ValidateScript({ Test-Path $_ })]
    [string]$CsvPath,

    [Parameter()]
    [string]$Domain = $env:USERDOMAIN,

    [Parameter()]
    [switch]$SendWelcomeEmail
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

# ── Logging ───────────────────────────────────────────
function Write-Log {
    param([string]$Message, [string]$Level = 'INFO')
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $entry = "[$timestamp] [$Level] $Message"
    Write-Host $entry -ForegroundColor $(switch ($Level) {
        'ERROR' { 'Red' }; 'WARN' { 'Yellow' }; default { 'Cyan' }
    })
    Add-Content -Path "$PSScriptRoot\script.log" -Value $entry
}

# ── Main ──────────────────────────────────────────────
try {
    Write-Log "Script started by $env:USERNAME"
    # ... logic here
    Write-Log "Script completed successfully"
}
catch {
    Write-Log "Fatal error: $_" -Level 'ERROR'
    exit 1
}
```

---

## Variables and Data Types

```powershell
# Scalars
[string]$Name    = "Alice"
[int]$Age        = 30
[double]$Pi      = 3.14159
[bool]$IsActive  = $true
[datetime]$Now   = Get-Date
[guid]$Id        = [guid]::NewGuid()

# Type casting
[int]"42"                   # 42
[string]42                  # "42"
[datetime]"2024-04-22"      # DateTime object

# Automatic variables
$true  / $false             # Boolean
$null                       # Null
$_   / $PSItem              # Current pipeline object
$?                          # Last command success (True/False)
$LASTEXITCODE               # Last native exit code
$PSVersionTable             # PowerShell version info
$PSScriptRoot               # Directory of running script
$MyInvocation               # Current script metadata
$args                       # Unnamed arguments array
$Error[0]                   # Last error

# Here-strings (multi-line)
$HTML = @"
<html>
    <body>Hello, $Name</body>
</html>
"@

$Literal = @'
No $variable expansion here.
Everything is literal.
'@
```

### Arrays and Collections

```powershell
# Array
$Fruits = @("apple", "banana", "cherry")
$Fruits += "date"                       # Append
$Fruits[0]                              # First element
$Fruits[-1]                             # Last element
$Fruits[1..3]                           # Slice
$Fruits.Count                           # Length
$Fruits | Sort-Object
$Fruits | Where-Object { $_ -like "b*" }
$Fruits -contains "apple"               # $true
$Fruits -notcontains "grape"            # $true

# ArrayList (mutable, faster for additions)
$List = [System.Collections.ArrayList]@()
[void]$List.Add("item1")               # [void] suppresses output
[void]$List.Add("item2")
$List.Remove("item1")
$List.Count

# Generic List (type-safe)
$TypedList = [System.Collections.Generic.List[string]]::new()
$TypedList.Add("item")

# Hashtable (key-value)
$Config = @{
    Host     = "localhost"
    Port     = 5432
    Database = "myapp"
    MaxConn  = 100
}
$Config["Host"]                         # Access by key
$Config.Port                            # Dot notation
$Config.Keys                            # All keys
$Config.Values                          # All values
$Config.ContainsKey("Port")             # $true
$Config.Remove("MaxConn")
$Config["Timeout"] = 30                 # Add/update

# Ordered hashtable (preserves insertion order)
$Ordered = [ordered]@{
    First  = 1
    Second = 2
    Third  = 3
}

# Array of hashtables (common pattern)
$Users = @(
    @{ Name = "Alice"; Role = "Admin" },
    @{ Name = "Bob";   Role = "User"  }
)
$Users | ForEach-Object { Write-Host "$($_.Name) is a $($_.Role)" }
```

---

## Control Flow

### Conditionals

```powershell
# if / elseif / else
if ($Age -ge 18) {
    Write-Host "Adult"
} elseif ($Age -ge 13) {
    Write-Host "Teenager"
} else {
    Write-Host "Child"
}

# Comparison operators
-eq   # Equal                    5 -eq 5
-ne   # Not equal                5 -ne 6
-gt   # Greater than             5 -gt 3
-lt   # Less than                3 -lt 5
-ge   # Greater or equal         5 -ge 5
-le   # Less or equal            5 -le 5
-like # Wildcard match           "Alice" -like "A*"
-notlike
-match   # Regex match           "hello123" -match "\d+"
-notmatch
-contains   # Array contains     @(1,2,3) -contains 2
-in         # Value in array     2 -in @(1,2,3)
-is         # Type check         "text" -is [string]
-and  -or  -not  -xor           # Logical operators

# Switch statement
switch ($Day) {
    "Monday"  { Write-Host "Start of week" }
    "Friday"  { Write-Host "End of week" }
    { $_ -in @("Saturday","Sunday") } { Write-Host "Weekend" }
    default   { Write-Host "Midweek" }
}

# Switch with regex
switch -Regex ($Input) {
    "^\d{4}-\d{2}-\d{2}$" { Write-Host "Date format" }
    "^\d+$"               { Write-Host "Integer" }
    "^[A-Za-z]+$"         { Write-Host "Alpha string" }
    default               { Write-Host "Unknown format" }
}
```

### Loops

```powershell
# foreach
foreach ($User in $Users) {
    Write-Host "Processing: $($User.Name)"
}

# ForEach-Object (pipeline)
Get-Process | ForEach-Object {
    if ($_.CPU -gt 100) {
        Write-Host "High CPU: $($_.Name)"
    }
}

# ForEach-Object with -Parallel (PowerShell 7+)
1..10 | ForEach-Object -Parallel {
    Start-Sleep -Milliseconds 100
    "Processed: $_"
} -ThrottleLimit 5

# for
for ($i = 0; $i -lt 10; $i++) {
    Write-Host "Item $i"
}

# while
$Attempts = 0
while ($Attempts -lt 3) {
    try {
        # attempt something
        break
    } catch {
        $Attempts++
        Start-Sleep -Seconds 5
    }
}

# do-while (runs at least once)
do {
    $Input = Read-Host "Enter 'yes' to continue"
} while ($Input -ne "yes")

# do-until
do {
    $Status = Get-Service "nginx"
    Start-Sleep -Seconds 2
} until ($Status.Status -eq "Running")

# Loop control
foreach ($Item in $Items) {
    if ($Item -eq "skip") { continue }
    if ($Item -eq "stop") { break }
    Write-Host $Item
}
```

---

## Functions

```powershell
# Basic function
function Get-Greeting {
    param(
        [Parameter(Mandatory)]
        [string]$Name,

        [Parameter()]
        [ValidateSet("Hello", "Hi", "Hey")]
        [string]$Salutation = "Hello",

        [Parameter()]
        [switch]$Loud
    )

    $Message = "$Salutation, $Name!"
    if ($Loud) { $Message = $Message.ToUpper() }
    return $Message
}

# Call
Get-Greeting -Name "Alice"
Get-Greeting -Name "Bob" -Salutation "Hi" -Loud
"Alice", "Bob" | ForEach-Object { Get-Greeting -Name $_ }

# Advanced function with pipeline support
function Format-FileSize {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, ValueFromPipeline, ValueFromPipelineByPropertyName)]
        [long]$Length
    )

    process {
        switch ($Length) {
            { $_ -ge 1GB } { "{0:N2} GB" -f ($_ / 1GB); break }
            { $_ -ge 1MB } { "{0:N2} MB" -f ($_ / 1MB); break }
            { $_ -ge 1KB } { "{0:N2} KB" -f ($_ / 1KB); break }
            default        { "$_ bytes" }
        }
    }
}

# Usage: Get-ChildItem | Select-Object Name, Length | ForEach-Object { $_.Length | Format-FileSize }

# Function with output object
function Get-ServerInfo {
    [CmdletBinding()]
    param([string]$ComputerName = $env:COMPUTERNAME)

    $OS  = Get-CimInstance Win32_OperatingSystem -ComputerName $ComputerName
    $CPU = Get-CimInstance Win32_Processor -ComputerName $ComputerName
    $Mem = [math]::Round($OS.TotalVisibleMemorySize / 1MB, 2)

    [PSCustomObject]@{
        ComputerName = $ComputerName
        OS           = $OS.Caption
        CPU          = $CPU.Name
        TotalRAM_GB  = $Mem
        FreeRAM_GB   = [math]::Round($OS.FreePhysicalMemory / 1MB, 2)
        Uptime       = (Get-Date) - $OS.LastBootUpTime
    }
}

Get-ServerInfo
Get-ServerInfo -ComputerName "SERVER01"
```

---

## Error Handling

```powershell
# Try / Catch / Finally
try {
    $Content = Get-Content "C:\missing.txt" -ErrorAction Stop
    Write-Host "File read successfully"
}
catch [System.IO.FileNotFoundException] {
    Write-Warning "File not found: $_"
}
catch [System.UnauthorizedAccessException] {
    Write-Error "Access denied: $_"
}
catch {
    Write-Error "Unexpected error: $($_.Exception.Message)"
    Write-Error "Stack trace: $($_.ScriptStackTrace)"
}
finally {
    Write-Host "Cleanup (always runs)"
}

# ErrorAction parameter
Get-Item "missing" -ErrorAction SilentlyContinue   # Ignore error
Get-Item "missing" -ErrorAction Stop               # Throw terminating error
Get-Item "missing" -ErrorAction Continue           # Default: show + continue
Get-Item "missing" -ErrorAction Inquire            # Ask user
$result = Get-Item "missing" -ErrorVariable myErr  # Store in variable
if ($myErr) { Write-Host "Error was: $myErr" }

# $ErrorActionPreference (script-wide default)
$ErrorActionPreference = 'Stop'     # All errors become terminating

# Retry pattern
function Invoke-WithRetry {
    param(
        [scriptblock]$ScriptBlock,
        [int]$MaxAttempts = 3,
        [int]$DelaySeconds = 5
    )
    $attempt = 1
    while ($attempt -le $MaxAttempts) {
        try {
            return & $ScriptBlock
        } catch {
            if ($attempt -eq $MaxAttempts) { throw }
            Write-Warning "Attempt $attempt failed. Retrying in ${DelaySeconds}s..."
            Start-Sleep -Seconds $DelaySeconds
            $attempt++
            $DelaySeconds *= 2    # Exponential backoff
        }
    }
}

Invoke-WithRetry -ScriptBlock {
    Invoke-RestMethod -Uri "https://api.example.com/data"
} -MaxAttempts 3
```

---

## Working with Files and Data

```powershell
# File I/O
Get-Content file.txt                            # Read all lines (array)
Get-Content file.txt -Raw                       # Read as single string
Get-Content file.txt | Select-Object -First 10  # Head
Get-Content file.txt -Tail 20                   # Tail
Get-Content file.txt -Wait                      # Follow (tail -f)
Set-Content file.txt "content"                  # Overwrite
Add-Content file.txt "new line"                 # Append
Out-File file.txt                               # Redirect output

# CSV
Import-Csv users.csv
Import-Csv users.csv | Where-Object { $_.Department -eq "IT" }
Export-Csv users.csv -NoTypeInformation         # Export without #TYPE header

# JSON
Get-Content config.json | ConvertFrom-Json
$Object | ConvertTo-Json -Depth 5
$Object | ConvertTo-Json | Out-File config.json

# XML
[xml]$Config = Get-Content config.xml
$Config.configuration.appSettings.add | Where-Object key -eq "ApiUrl"

# Registry
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion"
Set-ItemProperty "HKCU:\Software\MyApp" -Name "Theme" -Value "Dark"
New-Item "HKCU:\Software\MyApp" -Force

# Temp files
$TempFile = [System.IO.Path]::GetTempFileName()
$TempDir  = [System.IO.Path]::GetTempPath()
try {
    # use temp file
} finally {
    Remove-Item $TempFile -Force -ErrorAction SilentlyContinue
}
```

---

## Pipeline and Object Manipulation

```powershell
# Core pipeline cmdlets
Get-Process |
    Where-Object { $_.CPU -gt 50 } |              # Filter
    Select-Object Name, Id, CPU, WorkingSet |      # Project columns
    Sort-Object CPU -Descending |                  # Sort
    Select-Object -First 10 |                      # Limit
    Format-Table -AutoSize                         # Display

# Group-Object
Get-Process | Group-Object -Property Company | Sort-Object Count -Descending

# Measure-Object
Get-ChildItem -Recurse | Measure-Object -Property Length -Sum -Average -Maximum

# Calculated properties
Get-Process | Select-Object Name,
    @{Name="CPU_Pct"; Expression={ [math]::Round($_.CPU, 2) }},
    @{Name="RAM_MB";  Expression={ [math]::Round($_.WorkingSet / 1MB, 1) }}

# Expand nested properties
Get-ADUser -Filter * -Properties MemberOf |
    Select-Object SamAccountName, @{
        Name="Groups"; Expression={ $_.MemberOf -join ", " }
    }

# Tee-Object (output to screen AND file simultaneously)
Get-Process | Tee-Object -FilePath processes.txt | Where-Object CPU -gt 10

# ForEach-Object with Begin/Process/End
1..100 | ForEach-Object -Begin {
    $Sum = 0
    Write-Host "Starting..."
} -Process {
    $Sum += $_
} -End {
    Write-Host "Total: $Sum"
}
```

---

## Modules

```powershell
# Find and install modules
Find-Module -Name PSReadLine
Install-Module -Name Az -Scope CurrentUser -Force
Install-Module -Name ImportExcel -Scope CurrentUser

# List modules
Get-Module -ListAvailable
Get-Module                          # Currently loaded

# Import
Import-Module ActiveDirectory
Import-Module Az.Compute

# Create your own module
# File: MyTools.psm1
function Get-DiskUsage {
    Get-PSDrive -PSProvider FileSystem |
        Select-Object Name,
            @{N="Used_GB"; E={[math]::Round(($_.Used/1GB),2)}},
            @{N="Free_GB"; E={[math]::Round(($_.Free/1GB),2)}}
}
Export-ModuleMember -Function Get-DiskUsage

# Install personal module
# Copy MyTools.psm1 to: $env:USERPROFILE\Documents\PowerShell\Modules\MyTools\MyTools.psm1
Import-Module MyTools
Get-DiskUsage
```

---

## Remote Execution (PSRemoting)

```powershell
# Enable PSRemoting (run as admin on target)
Enable-PSRemoting -Force

# One-to-one remote session
Enter-PSSession -ComputerName SERVER01
Enter-PSSession -ComputerName SERVER01 -Credential (Get-Credential)

# One-to-many (run on multiple servers)
$Servers = @("WEB01", "WEB02", "WEB03", "DB01")

Invoke-Command -ComputerName $Servers -ScriptBlock {
    Get-Service nginx | Select-Object Name, Status
}

# Parallel execution with throttle
Invoke-Command -ComputerName $Servers -ThrottleLimit 10 -ScriptBlock {
    [PSCustomObject]@{
        Server  = $env:COMPUTERNAME
        CPU     = (Get-CimInstance Win32_Processor).LoadPercentage
        FreeMem = [math]::Round((Get-CimInstance Win32_OperatingSystem).FreePhysicalMemory / 1MB, 2)
        Disk    = (Get-PSDrive C).Free / 1GB
    }
}

# Persistent session (reuse connection)
$Session = New-PSSession -ComputerName SERVER01
Invoke-Command -Session $Session -ScriptBlock { $env:COMPUTERNAME }
Copy-Item -Path "C:\deploy\app.zip" -Destination "C:\apps\" -ToSession $Session
Remove-PSSession -Session $Session

# SSH-based remoting (cross-platform, PowerShell 7+)
Enter-PSSession -HostName ubuntu@192.168.1.100 -SSHTransport
Invoke-Command -HostName ubuntu@192.168.1.100 -SSHTransport -ScriptBlock { uname -a }
```

---

## Practical Scripts

### Bulk Active Directory User Creation

```powershell
# newusers.csv:
# FirstName,LastName,Department,Manager
# Alice,Smith,IT,john.doe
# Bob,Jones,Finance,jane.doe

Import-Module ActiveDirectory

$Users = Import-Csv ".\newusers.csv"
$Domain = "corp.example.com"
$DefaultOU = "OU=NewUsers,DC=corp,DC=example,DC=com"
$DefaultPassword = ConvertTo-SecureString "Welcome@2024!" -AsPlainText -Force

foreach ($User in $Users) {
    $SamAccount  = "$($User.FirstName.ToLower()).$($User.LastName.ToLower())"
    $UPN         = "$SamAccount@$Domain"
    $DisplayName = "$($User.FirstName) $($User.LastName)"

    try {
        if (Get-ADUser -Filter { SamAccountName -eq $SamAccount } -ErrorAction SilentlyContinue) {
            Write-Warning "User already exists: $SamAccount"
            continue
        }

        New-ADUser `
            -SamAccountName    $SamAccount `
            -UserPrincipalName $UPN `
            -GivenName         $User.FirstName `
            -Surname           $User.LastName `
            -DisplayName       $DisplayName `
            -Department        $User.Department `
            -Manager           $User.Manager `
            -Path              $DefaultOU `
            -AccountPassword   $DefaultPassword `
            -ChangePasswordAtLogon $true `
            -Enabled           $true

        Write-Host "Created: $DisplayName ($SamAccount)" -ForegroundColor Green
    }
    catch {
        Write-Error "Failed to create $SamAccount : $_"
    }
}
```

### Server Health Report

```powershell
$Servers = Get-Content ".\servers.txt"
$Report  = [System.Collections.Generic.List[PSObject]]::new()

$Results = Invoke-Command -ComputerName $Servers -ErrorAction SilentlyContinue -ScriptBlock {
    $OS  = Get-CimInstance Win32_OperatingSystem
    $CPU = (Get-CimInstance Win32_Processor | Measure-Object LoadPercentage -Average).Average
    $Disk = Get-PSDrive C

    [PSCustomObject]@{
        Server      = $env:COMPUTERNAME
        OS          = $OS.Caption
        Uptime_Days = [math]::Round(((Get-Date) - $OS.LastBootUpTime).TotalDays, 1)
        CPU_Pct     = $CPU
        RAM_Free_GB = [math]::Round($OS.FreePhysicalMemory / 1MB, 2)
        Disk_Free_GB= [math]::Round($Disk.Free / 1GB, 2)
        Disk_Pct    = [math]::Round((($Disk.Used / ($Disk.Used + $Disk.Free)) * 100), 1)
        Status      = "OK"
    }
}

$Results | Sort-Object CPU_Pct -Descending |
    Format-Table -AutoSize

# Export to Excel (requires ImportExcel module)
$Results | Export-Excel "ServerHealth_$(Get-Date -f yyyyMMdd).xlsx" `
    -AutoSize -BoldTopRow -FreezeTopRow `
    -ConditionalText (
        New-ConditionalText -Text "OK"    -ConditionalTextColor Green,
        New-ConditionalText -Range "E:E" -RuleType GreaterThan -ConditionValue 80 -BackgroundColor Red
    )
```

### Scheduled Task via Script

```powershell
# Register a scheduled task to run a PS script daily
$Action  = New-ScheduledTaskAction -Execute "pwsh.exe" `
               -Argument "-NonInteractive -File C:\Scripts\backup.ps1"
$Trigger = New-ScheduledTaskTrigger -Daily -At "02:00AM"
$Settings= New-ScheduledTaskSettingsSet `
               -ExecutionTimeLimit (New-TimeSpan -Hours 2) `
               -RestartCount 3 `
               -RestartInterval (New-TimeSpan -Minutes 10)
$Principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -RunLevel Highest

Register-ScheduledTask `
    -TaskName    "DailyBackup" `
    -TaskPath    "\MyScripts\" `
    -Action      $Action `
    -Trigger     $Trigger `
    -Settings    $Settings `
    -Principal   $Principal `
    -Description "Daily system backup"

# Run immediately
Start-ScheduledTask -TaskName "\MyScripts\DailyBackup"
```

---

## PowerShell Profiles

```powershell
# Profile locations
$PROFILE                            # Current user, current host
$PROFILE.AllUsersAllHosts           # All users, all hosts
$PROFILE.CurrentUserAllHosts        # Current user, all hosts

# Edit profile
notepad $PROFILE
code $PROFILE                       # VS Code

# Example ~/.config/powershell/Microsoft.PowerShell_profile.ps1
Set-PSReadLineOption -EditMode Emacs
Set-PSReadLineKeyHandler -Key Tab -Function MenuComplete
Set-Alias -Name ll -Value Get-ChildItem
Set-Alias -Name grep -Value Select-String

function prompt {
    $branch = git branch --show-current 2>$null
    $gitPart = if ($branch) { " [$branch]" } else { "" }
    "PS $($PWD.Path)$gitPart> "
}

Import-Module posh-git
Import-Module PSReadLine
```

---

## Related Topics

- [Windows CLI ←](04_Windows_CLI.md) — cmdlet basics
- [Active Directory ←](09_Active_Directory.md) — AD automation
- [Services & Processes ←](15_Services_Processes.md) — service management
- [CI/CD ←](27_CICD_Fundamentals.md) — PS scripts in pipelines
- [Monitoring & Logging ←](13_Monitoring_Logging.md) — event log queries

---

> [Index](00_INDEX.md)
