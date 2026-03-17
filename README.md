# MRrobot-MEWIZAR-DEFENCE-
NOW HOW TO DEFEND AGAINST IT ALL 
<#
.SYNOPSIS
    MR. ROBOT DEFENDER – Ultimate Defense & Detection Toolkit
    Monitors for attacks, blocks threats, logs everything.
.NOTES
    Run as Administrator. Requires Windows 10/11 Pro/Enterprise for some features.
#>

#requires -RunAsAdministrator

# ========== Configuration ==========
$DEFENSE_LOG = "$env:USERPROFILE\Desktop\MR_ROBOT_DEFENSE_LOG_$(Get-Date -Format 'yyyyMMdd').txt"
$BLOCKED_IPS_LOG = "$env:USERPROFILE\Desktop\blocked_ips.txt"
$ALERT_EMAIL = "admin@yourdomain.com"  # Change to your email for alerts
$ALERT_SOUND = $true  # Play sound on alert

# ========== Color Output ==========
$RED = 'Red'
$GREEN = 'Green'
$YELLOW = 'Yellow'
$CYAN = 'Cyan'
$MAGENTA = 'Magenta'

# ========== Helper Functions ==========
function Write-Log {
    param([string]$Message, [string]$Color = "White")
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $logEntry = "[$timestamp] $Message"
    Add-Content -Path $DEFENSE_LOG -Value $logEntry
    Write-Host $logEntry -ForegroundColor $Color
    if ($ALERT_SOUND -and $Color -eq "Red") {
        [System.Console]::Beep(1000, 500)
    }
}

function Test-PortOpen {
    param([string]$IP, [int]$Port)
    try {
        $tcp = New-Object System.Net.Sockets.TcpClient
        $connect = $tcp.BeginConnect($IP, $Port, $null, $null)
        $wait = $connect.AsyncWaitHandle.WaitOne(500, $false)
        if ($wait -and $tcp.Connected) {
            $tcp.Close()
            return $true
        }
    } catch {}
    return $false
}

# ========== 1. Detect & Block Tor/Proxy Usage ==========
function Detect-TorProxies {
    Write-Log "🔍 Scanning for Tor/proxy services..." -Color $CYAN
    
    $torPorts = @(9050, 9051, 9150, 9151, 8118, 8123, 9999, 5566, 4444, 17333)
    $suspiciousIPs = @()
    
    # Check for Tor exit nodes in firewall (requires external API - we'll use local heuristics)
    Write-Log "  Checking for Tor/proxy ports..." -Color $YELLOW
    
    $connections = Get-NetTCPConnection -ErrorAction SilentlyContinue | Where-Object { 
        $_.RemotePort -in $torPorts -or $_.LocalPort -in $torPorts 
    }
    
    if ($connections) {
        Write-Log "  ⚠️ TOR/PROXY DETECTED!" -Color $RED
        $connections | ForEach-Object {
            Write-Log "    $($_.LocalAddress):$($_.LocalPort) ↔ $($_.RemoteAddress):$($_.RemotePort)" -Color $RED
            $suspiciousIPs += $_.RemoteAddress
        }
        
        # Block the IPs
        foreach ($ip in $suspiciousIPs | Select-Object -Unique) {
            try {
                $ruleName = "Block_Tor_$ip"
                New-NetFirewallRule -DisplayName $ruleName -Direction Outbound -RemoteAddress $ip -Action Block | Out-Null
                Write-Log "    🚫 Blocked outbound to $ip" -Color $GREEN
            } catch {
                Write-Log "    Failed to block $ip : $_" -Color $RED
            }
        }
    } else {
        Write-Log "  ✓ No Tor/proxy services detected" -Color $GREEN
    }
    
    # Check for Tor Browser installation
    $torPaths = @(
        "$env:ProgramFiles\Tor Browser",
        "$env:LOCALAPPDATA\Tor Browser",
        "$env:ProgramData\chocolatey\lib\tor"
    )
    
    foreach ($path in $torPaths) {
        if (Test-Path $path) {
            Write-Log "  ⚠️ Tor Browser installation found at: $path" -Color $YELLOW
        }
    }
}

# ========== 2. Monitor Printer Attacks (Port 9100) ==========
function Monitor-Printers {
    Write-Log "🖨️ Monitoring printer ports (9100)..." -Color $CYAN
    
    # Get all TCP connections on port 9100
    $printerConnections = Get-NetTCPConnection -ErrorAction SilentlyContinue | Where-Object { 
        $_.LocalPort -eq 9100 -or $_.RemotePort -eq 9100 
    }
    
    if ($printerConnections) {
        Write-Log "  ⚠️ ACTIVE PRINTER CONNECTIONS DETECTED!" -Color $RED
        $printerConnections | ForEach-Object {
            Write-Log "    $($_.LocalAddress):9100 ↔ $($_.RemoteAddress)" -Color $RED
        }
        
        # Alert (could send email)
        Write-Log "  Printer activity logged - check for malicious print jobs" -Color $YELLOW
    } else {
        Write-Log "  ✓ No active printer connections" -Color $GREEN
    }
    
    # Check for large print spooler files (potential DoS)
    $spoolPath = "C:\Windows\System32\spool\PRINTERS"
    if (Test-Path $spoolPath) {
        $largeFiles = Get-ChildItem $spoolPath -ErrorAction SilentlyContinue | Where-Object { $_.Length -gt 10MB }
        if ($largeFiles) {
            Write-Log "  ⚠️ Large spool files found (possible print bomb)" -Color $RED
            $largeFiles | ForEach-Object { Write-Log "    $($_.Name) - $($_.Length) bytes" -Color $YELLOW }
        }
    }
}

# ========== 3. Detect GitHub Cloning / Data Exfiltration ==========
function Detect-GitExfiltration {
    Write-Log "📡 Monitoring for data exfiltration..." -Color $CYAN
    
    # Look for git processes cloning repos
    $gitProcs = Get-Process -Name "git" -ErrorAction SilentlyContinue
    if ($gitProcs) {
        Write-Log "  ⚠️ Git processes running - possible cloning!" -Color $RED
        $gitProcs | ForEach-Object {
            $cmdLine = (Get-WmiObject Win32_Process -Filter "ProcessId=$($_.Id)").CommandLine
            Write-Log "    PID $($_.Id): $cmdLine" -Color $YELLOW
        }
    }
    
    # Monitor for large outbound transfers
    $connections = Get-NetTCPConnection -ErrorAction SilentlyContinue | Where-Object { 
        $_.State -eq "Established" -and $_.RemotePort -in @(22, 443, 80, 9418)  # SSH, HTTPS, HTTP, Git
    }
    
    $highDataIPs = @()
    foreach ($conn in $connections) {
        # This is simplified - real detection would use netstat counters
        if ($conn.RemoteAddress -notmatch "^(192\.168\.|10\.|172\.(1[6-9]|2[0-9]|3[01])\.)") {
            Write-Log "  ⚠️ External connection to $($conn.RemoteAddress):$($conn.RemotePort)" -Color $YELLOW
            $highDataIPs += $conn.RemoteAddress
        }
    }
    
    if ($highDataIPs.Count -gt 5) {
        Write-Log "  ⚠️ MULTIPLE EXTERNAL CONNECTIONS - POSSIBLE EXFILTRATION!" -Color $RED
    }
}

# ========== 4. Detect Anonymous Account Creation ==========
function Detect-AnonymousAccounts {
    Write-Log "👤 Checking for anonymous/local accounts..." -Color $CYAN
    
    $localUsers = Get-LocalUser | Where-Object { 
        $_.Name -like "User_*" -or 
        $_.Name -like "anon*" -or
        $_.Name -like "temp*" -or
        $_.Description -like "*Auto-generated*"
    }
    
    if ($localUsers) {
        Write-Log "  ⚠️ SUSPICIOUS LOCAL ACCOUNTS FOUND!" -Color $RED
        $localUsers | ForEach-Object {
            Write-Log "    $($_.Name) - $($_.Description)" -Color $RED
        }
    } else {
        Write-Log "  ✓ No suspicious local accounts" -Color $GREEN
    }
    
    # Check for recently created accounts (last 24h)
    $recentUsers = Get-WinEvent -FilterHashtable @{LogName='Security'; ID=4720; StartTime=(Get-Date).AddDays(-1)} -ErrorAction SilentlyContinue
    if ($recentUsers) {
        Write-Log "  ⚠️ NEW ACCOUNTS CREATED IN LAST 24H!" -Color $RED
        $recentUsers | ForEach-Object {
            Write-Log "    $($_.Message)" -Color $YELLOW
        }
    }
}

# ========== 5. Monitor Scheduled Tasks ==========
function Detect-ScheduledTasks {
    Write-Log "⏰ Scanning scheduled tasks..." -Color $CYAN
    
    $suspiciousTasks = Get-ScheduledTask | Where-Object { 
        $_.TaskName -like "*MRROBOT*" -or
        $_.TaskName -like "*Random*" -or
        $_.TaskName -like "*Miner*" -or
        $_.TaskName -like "*Crypto*"
    }
    
    if ($suspiciousTasks) {
        Write-Log "  ⚠️ SUSPICIOUS SCHEDULED TASKS FOUND!" -Color $RED
        $suspiciousTasks | ForEach-Object {
            Write-Log "    $($_.TaskName) - Next run: $($_.NextRunTime)" -Color $RED
            
            # Disable the task
            try {
                Disable-ScheduledTask -TaskName $_.TaskName
                Write-Log "      🚫 Task disabled" -Color $GREEN
            } catch {
                Write-Log "      Failed to disable" -Color $RED
            }
        }
    } else {
        Write-Log "  ✓ No suspicious scheduled tasks" -Color $GREEN
    }
}

# ========== 6. Detect Bitcoin Miners ==========
function Detect-BitcoinMiners {
    Write-Log "⛏️ Scanning for cryptocurrency miners..." -Color $CYAN
    
    $minerProcesses = @("xmrig", "miner", "ccminer", "ethminer", "claymore", "nicehash", "cgminer", "bfgminer")
    $found = $false
    
    foreach ($proc in $minerProcesses) {
        $running = Get-Process -Name $proc -ErrorAction SilentlyContinue
        if ($running) {
            Write-Log "  ⚠️ MINER PROCESS DETECTED: $proc" -Color $RED
            $running | ForEach-Object {
                Write-Log "    PID $($_.Id) - CPU: $($_.CPU)" -Color $RED
                try {
                    Stop-Process -Id $_.Id -Force
                    Write-Log "      🚫 Process terminated" -Color $GREEN
                } catch {
                    Write-Log "      Failed to terminate" -Color $RED
                }
            }
            $found = $true
        }
    }
    
    if (-not $found) {
        Write-Log "  ✓ No miners detected" -Color $GREEN
    }
    
    # Check for miner files
    $minerFiles = Get-ChildItem -Path "$env:USERPROFILE" -Filter "*.exe" -Recurse -ErrorAction SilentlyContinue | 
                  Where-Object { $_.Name -match "xmrig|miner|ccminer" }
    if ($minerFiles) {
        Write-Log "  ⚠️ MINER EXECUTABLES FOUND!" -Color $RED
        $minerFiles | ForEach-Object {
            Write-Log "    $($_.FullName)" -Color $RED
        }
    }
}

# ========== 7. Monitor for Network Scans ==========
function Detect-NetworkScans {
    Write-Log "🔎 Detecting network scanning..." -Color $CYAN
    
    # Look for nmap/masscan processes
    $scanTools = @("nmap", "masscan", "zenmap", "angryip")
    foreach ($tool in $scanTools) {
        $running = Get-Process -Name $tool -ErrorAction SilentlyContinue
        if ($running) {
            Write-Log "  ⚠️ SCAN TOOL DETECTED: $tool" -Color $RED
            $running | ForEach-Object {
                Write-Log "    PID $($_.Id) - scanning in progress" -Color $RED
                try {
                    Stop-Process -Id $_.Id -Force
                    Write-Log "      🚫 Scanner terminated" -Color $GREEN
                } catch {
                    Write-Log "      Failed to terminate" -Color $RED
                }
            }
        }
    }
    
    # Look for rapid connection attempts (port scan)
    $recentConnections = Get-NetTCPConnection -ErrorAction SilentlyContinue | 
                         Group-Object RemoteAddress | 
                         Where-Object { $_.Count -gt 20 }
    
    if ($recentConnections) {
        Write-Log "  ⚠️ RAPID CONNECTIONS FROM SINGLE IP - POSSIBLE PORT SCAN!" -Color $RED
        $recentConnections | ForEach-Object {
            Write-Log "    $($_.Name) - $($_.Count) connections" -Color $RED
            
            # Block the scanning IP
            try {
                $ruleName = "Block_Scanner_$($_.Name)"
                New-NetFirewallRule -DisplayName $ruleName -Direction Inbound -RemoteAddress $($_.Name) -Action Block | Out-Null
                Add-Content -Path $BLOCKED_IPS_LOG -Value "$(Get-Date) - Blocked scanner: $($_.Name)"
                Write-Log "      🚫 IP blocked" -Color $GREEN
            } catch {
                Write-Log "      Failed to block" -Color $RED
            }
        }
    }
}

# ========== 8. Monitor for PowerShell Abuse ==========
function Detect-PowerShellAbuse {
    Write-Log "💻 Monitoring for PowerShell abuse..." -Color $CYAN
    
    # Look for PowerShell processes with suspicious arguments
    $psProcs = Get-Process -Name "powershell", "pwsh" -ErrorAction SilentlyContinue
    foreach ($proc in $psProcs) {
        $cmdLine = (Get-WmiObject Win32_Process -Filter "ProcessId=$($proc.Id)" -ErrorAction SilentlyContinue).CommandLine
        if ($cmdLine -match "-EncodedCommand" -or $cmdLine -match "iex" -or $cmdLine -match "DownloadString") {
            Write-Log "  ⚠️ SUSPICIOUS POWERSHELL COMMAND!" -Color $RED
            Write-Log "    $cmdLine" -Color $RED
            try {
                Stop-Process -Id $proc.Id -Force
                Write-Log "      🚫 Process terminated" -Color $GREEN
            } catch {
                Write-Log "      Failed to terminate" -Color $RED
            }
        }
    }
}

# ========== 9. Enable Windows Defender Hardening ==========
function Enable-DefenderHardening {
    Write-Log "🛡️ Hardening Windows Defender..." -Color $CYAN
    
    try {
        # Enable cloud protection
        Set-MpPreference -MAPSReporting Advanced
        Set-MpPreference -SubmitSamplesConsent SendSafeSamplesAutomatically
        
        # Enable PUA protection
        Set-MpPreference -PUAProtection Enabled
        
        # Enable network protection
        Set-MpPreference -EnableNetworkProtection Enabled
        
        # Set high scan level
        Set-MpPreference -ScanAvgCPULoadFactor 50
        Set-MpPreference -RealTimeScanDirection Incoming
        
        Write-Log "  ✓ Windows Defender hardened" -Color $GREEN
    } catch {
        Write-Log "  Failed to harden Defender: $_" -Color $RED
    }
}

# ========== 10. Create Persistent Monitoring Service ==========
function Install-MonitoringService {
    Write-Log "🔧 Installing persistent monitoring service..." -Color $CYAN
    
    $serviceScript = @"
`$logFile = "C:\ProgramData\MRRobotMonitor.log"
while(`$true) {
    `$timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    
    # Check for suspicious processes every minute
    `$badProcs = @("nmap", "masscan", "xmrig", "miner", "hydra", "sqlmap", "nikto")
    foreach (`$p in `$badProcs) {
        `$running = Get-Process -Name `$p -ErrorAction SilentlyContinue
        if (`$running) {
            `$msg = "[`$timestamp] Killed malicious process: `$p"
            Add-Content -Path `$logFile -Value `$msg
            Stop-Process -Name `$p -Force
        }
    }
    
    # Check for Tor ports
    `$torPorts = @(9050,9051,9150)
    `$conn = Get-NetTCPConnection -ErrorAction SilentlyContinue | Where-Object { `$_.RemotePort -in `$torPorts }
    if (`$conn) {
        `$msg = "[`$timestamp] Tor connection blocked: `$(`$conn.RemoteAddress):`$(`$conn.RemotePort)"
        Add-Content -Path `$logFile -Value `$msg
        # Block IP via firewall (would need admin)
    }
    
    Start-Sleep -Seconds 60
}
"@
    
    $serviceScriptPath = "C:\ProgramData\MRRobotMonitor.ps1"
    $serviceScript | Out-File -FilePath $serviceScriptPath -Encoding utf8
    
    # Create scheduled task to run at startup
    $taskName = "MRRobotDefender"
    $action = New-ScheduledTaskAction -Execute "PowerShell.exe" -Argument "-NoProfile -WindowStyle Hidden -File `"$serviceScriptPath`""
    $trigger = New-ScheduledTaskTrigger -AtStartup
    $principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -LogonType ServiceAccount -RunLevel Highest
    $settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -StartWhenAvailable -RunOnlyIfNetworkAvailable -MultipleInstances IgnoreNew
    
    try {
        Register-ScheduledTask -TaskName $taskName -Action $action -Trigger $trigger -Principal $principal -Settings $settings -Force
        Write-Log "  ✓ Persistent monitoring installed (runs at startup)" -Color $GREEN
    } catch {
        Write-Log "  Failed to install monitoring: $_" -Color $RED
    }
}

# ========== 11. Create Emergency Block Script ==========
function Create-EmergencyBlock {
    Write-Log "🚨 Creating emergency block script..." -Color $CYAN
    
    $blockScript = @"
# Emergency Network Block – Run to block all external traffic
Write-Host "🚨 EMERGENCY BLOCK ACTIVATED" -ForegroundColor Red

# Block all outbound except local subnet
`$localIP = (Get-NetIPAddress -AddressFamily IPv4 | Where-Object { `$_.InterfaceAlias -notlike '*Loopback*' }).IPAddress | Select-Object -First 1
`$subnet = (`$localIP -split '\.')[0..2] -join '.' + '.0/24'

# Create firewall rules
New-NetFirewallRule -DisplayName "EMERGENCY_BLOCK_ALL" -Direction Outbound -Action Block
New-NetFirewallRule -DisplayName "EMERGENCY_ALLOW_LOCAL" -Direction Outbound -RemoteAddress `$subnet -Action Allow

Write-Host "✅ All external traffic blocked. Only local subnet (`$subnet) allowed." -ForegroundColor Green
"@
    
    $blockScript | Out-File -FilePath "$env:USERPROFILE\Desktop\EMERGENCY_BLOCK.ps1" -Encoding utf8
    Write-Log "  ✓ Emergency block script saved to Desktop" -Color $GREEN
}

# ========== MAIN EXECUTION ==========
Clear-Host
Write-Host @"

██████╗ ███████╗███████╗███████╗███╗   ██╗██████╗ ███████╗██████╗ 
██╔══██╗██╔════╝██╔════╝██╔════╝████╗  ██║██╔══██╗██╔════╝██╔══██╗
██║  ██║█████╗  █████╗  █████╗  ██╔██╗ ██║██║  ██║█████╗  ██████╔╝
██║  ██║██╔══╝  ██╔══╝  ██╔══╝  ██║╚██╗██║██║  ██║██╔══╝  ██╔══██╗
██████╔╝███████╗██║     ███████╗██║ ╚████║██████╔╝███████╗██║  ██║
╚═════╝ ╚══════╝╚═╝     ╚══════╝╚═╝  ╚═══╝╚═════╝ ╚══════╝╚═╝  ╚═╝
                                                                     
██╗  ██╗ ██████╗ ██████╗  █████╗ ████████╗    ██████╗ ███████╗███████╗███████╗███╗   ██╗██████╗ ███████╗██████╗ 
██║  ██║██╔═══██╗██╔══██╗██╔══██╗╚══██╔══╝    ██╔══██╗██╔════╝██╔════╝██╔════╝████╗  ██║██╔══██╗██╔════╝██╔══██╗
███████║██║   ██║██████╔╝███████║   ██║       ██║  ██║█████╗  █████╗  █████╗  ██╔██╗ ██║██║  ██║█████╗  ██████╔╝
██╔══██║██║   ██║██╔══██╗██╔══██║   ██║       ██║  ██║██╔══╝  ██╔══╝  ██╔══╝  ██║╚██╗██║██║  ██║██╔══╝  ██╔══██╗
██║  ██║╚██████╔╝██║  ██║██║  ██║   ██║       ██████╔╝███████╗██║     ███████╗██║ ╚████║██████╔╝███████╗██║  ██║
╚═╝  ╚═╝ ╚═════╝ ╚═╝  ╚═╝╚═╝  ╚═╝   ╚═╝       ╚═════╝ ╚══════╝╚═╝     ╚══════╝╚═╝  ╚═══╝╚═════╝ ╚══════╝╚═╝  ╚═╝
"@ -ForegroundColor $CYAN

Write-Host "`n                      MR. ROBOT DEFENDER – Ultimate Defense Toolkit" -ForegroundColor $MAGENTA
Write-Host "                          Protecting against the very attacks we built" -ForegroundColor $YELLOW
Write-Host "`nLogging to: $DEFENSE_LOG" -ForegroundColor $WHITE

Write-Log "====== MR. ROBOT DEFENDER STARTING ======" -Color $MAGENTA

# Run all detection modules
Detect-TorProxies
Monitor-Printers
Detect-GitExfiltration
Detect-AnonymousAccounts
Detect-ScheduledTasks
Detect-BitcoinMiners
Detect-NetworkScans
Detect-PowerShellAbuse
Enable-DefenderHardening
Install-MonitoringService
Create-EmergencyBlock

# ========== Final Summary ==========
Write-Host @"

╔═══════════════════════════════════════════════════════════════════╗
║                 MR. ROBOT DEFENDER – SUMMARY                      ║
╚═══════════════════════════════════════════════════════════════════╝

✅ All detection modules completed
✅ Windows Defender hardened
✅ Persistent monitoring installed (runs at startup)
✅ Emergency block script on Desktop

📁 DEFENSE LOG: $DEFENSE_LOG
📁 BLOCKED IPS: $BLOCKED_IPS_LOG
📁 EMERGENCY BLOCK: Desktop\EMERGENCY_BLOCK.ps1

🔹 TO VIEW ACTIVE THREATS:   Get-Content "$DEFENSE_LOG" -Tail 20
🔹 TO BLOCK ALL EXTERNAL:    Run Desktop\EMERGENCY_BLOCK.ps1
🔹 TO STOP MONITORING:       Unregister-ScheduledTask -TaskName "MRRobotDefender" -Confirm:$false

⚠️ This script only protects this machine – network‑wide protection requires
   enterprise firewall/IDS solutions.

Press Enter to exit...
"@ -ForegroundColor $CYAN
Read-Host
