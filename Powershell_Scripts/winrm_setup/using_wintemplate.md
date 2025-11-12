Below is your **updated, structured, and complete PowerShell workflow** that includes **setup + verification commands** for each section (WinRM, listeners, CredSSP, certificates, firewall, and network profile).

---

## 🧩 Full WinRM Setup & Verification Script

### 1️⃣ Check if WinRM is Configured

```powershell
# Check WinRM service status
Get-Service WinRM

# Check basic WinRM configuration
winrm get winrm/config

# Quick summary of WinRM listener configuration
winrm enumerate winrm/config/listener

# Check if WinRM is ready (should show "WinRM is already set up")
winrm quickconfig
```

---

### 2️⃣ Enable / Setup WinRM

```powershell
Enable-PSRemoting -Force
Set-WSManQuickConfig -Force
# Optional (HTTPS)
# winrm quickconfig -transport:https
```

---

### 3️⃣ Verify if a Listener Already Exists

```powershell
# List all existing listeners
winrm enumerate winrm/config/listener

# Or in PowerShell object format (easier to filter)
Get-ChildItem WSMan:\localhost\Listener | Select-Object -ExpandProperty Keys
```

If you see a line like:

```
Listener [Source="GPO"]
    Address = *
    Transport = HTTPS
```

then one already exists.

---

### 4️⃣ Check for Existing Certificates

```powershell
# Check if a certificate for this hostname already exists
$Hostname = [System.Net.Dns]::GetHostByName($env:computerName).HostName
Get-ChildItem Cert:\LocalMachine\My | Where-Object { $_.Subject -like "*CN=$Hostname*" } | 
Select-Object Subject, Thumbprint, NotAfter
```

---

### 5️⃣ Create New Self-Signed Certificate (if none exists)

```powershell
$Cert = New-SelfSignedCertificate -Subject "CN=$Hostname" 
-TextExtension '2.5.29.37={text}1.3.6.1.5.5.7.3.1' 
-CertStoreLocation Cert:\LocalMachine\My

$Thumbprint = $Cert.Thumbprint
Write-Host "Certificate created with thumbprint: $Thumbprint" -ForegroundColor Green
```

---

### 6️⃣ Create a Custom HTTPS Listener (if not already present)

```powershell
# Verify first(this can not be done like this view windows11lckdwn.md for correct way)
$existing = winrm enumerate winrm/config/listener | Select-String "HTTPS"
if (-not $existing) {
    winrm create winrm/config/Listener?Address=*+Transport=HTTPS "@{Hostname='$Hostname';CertificateThumbprint='$Thumbprint'}"
    Write-Host "New HTTPS listener created successfully!" -ForegroundColor Green
} else {
    Write-Host "An HTTPS listener already exists." -ForegroundColor Yellow
}
```

---

### 7️⃣ Check Firewall Rules

```powershell
# Check existing WinRM firewall rules
Get-NetFirewallRule -DisplayName "*WinRM*" | ft DisplayName, Enabled, Direction, Action

# Check if a rule for port 5986 exists
Get-NetFirewallPortFilter | Where-Object { $_.LocalPort -eq 5986 }

# Check if the ports is enabled
netstat -ano | findstr ":5986"


# If not found, create one
if (-not (Get-NetFirewallRule -DisplayName "Allow WinRM HTTPS" -ErrorAction SilentlyContinue)) {
    New-NetFirewallRule -Name "AllowWinRM_HTTPS" -DisplayName "Allow WinRM HTTPS" `
    -Enabled True -Direction Inbound -Protocol TCP -LocalPort 5986 -Action Allow
    Write-Host "Firewall rule for WinRM HTTPS created." -ForegroundColor Green
} else {
    Write-Host "Firewall rule for WinRM HTTPS already exists." -ForegroundColor Yellow
}
```

---

### 8️⃣ Enable and Verify CredSSP

```powershell
# Enable CredSSP for the server role
Enable-WSManCredSSP -Role Server -Force

# Ensure it's enabled in WSMan configuration
Set-Item -Path WSMan:\localhost\Service\Auth\CredSSP -Value $true

# Restart WinRM service to apply changes
Restart-Service WinRM

# Verify CredSSP status
Get-Item -Path WSMan:\localhost\Service\Auth\CredSSP
# Should return:
# Value : true
```

---

### 9️⃣ Check and Change Network Profile

```powershell
# List all network interfaces and profiles
Get-NetConnectionProfile | Select-Object Name, InterfaceIndex, NetworkCategory

# If network is Public, change it to Private
Set-NetConnectionProfile -InterfaceIndex YourInterfaceIndex -NetworkCategory Private

# Verify change
Get-NetConnectionProfile | Select-Object Name, InterfaceIndex, NetworkCategory
```

---

### 🔟 Optional: Full Health Check Summary (Run Anytime)

This gives you a single snapshot of the system’s remote management state.

```powershell
Write-Host "`n===== WinRM Health Check =====`n" -ForegroundColor Cyan
Get-Service WinRM
winrm enumerate winrm/config/listener
(Get-Item -Path WSMan:\localhost\Service\Auth\CredSSP).Value
Get-NetFirewallRule -DisplayName "*WinRM*" | ft DisplayName, Enabled
Get-ChildItem Cert:\LocalMachine\My | Where-Object { $_.Subject -like "*CN=$Hostname*" } | 
Select-Object Subject, Thumbprint, NotAfter
Get-NetConnectionProfile | Select-Object Name, InterfaceIndex, NetworkCategory
```

---

Would you like me to **bundle all of this into one reusable PowerShell script file** (`Setup-WinRM.ps1`) with proper logging and color-coded output (e.g., green for success, red for missing configs)?
That way you can just run it anytime to both configure and verify everything automatically.
