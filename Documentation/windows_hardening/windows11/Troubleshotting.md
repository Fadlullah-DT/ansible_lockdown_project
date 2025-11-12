### 2.3 Diagnosing ping / network issues

* From host and from server: ping to `192.168.0.183` → 100% packet loss.
* On Windows 11 VM: ping to the server succeeded.
  **Diagnosis:** A GPO from the domain‐controller had blocked ICMP (ping) inbound to the Windows 11 VM as part of the hardening baseline.
  **Fix applied:** On the Windows 11 VM I ran:

```powershell
netsh advfirewall firewall add rule name="Allow ICMPv4 In" dir=in action=allow protocol=icmpv4
```

After that, I could successfully ping the VM from both the host and server.

### 2.4 Checking WinRM firewall rule

On the VM I checked:

```powershell
Get-NetFirewallRule -DisplayName "Windows Remote Management (HTTP-In)"
```

If that rule is missing, add it:

```powershell
netsh advfirewall firewall add rule name="WinRM HTTP" dir=in action=allow protocol=TCP localport=5985
```

### 2.5 Check WinRM listener on VM

```powershell
winrm enumerate winrm/config/listener
```

Expected output shows something like:

``` powershell
Transport = HTTP  
Port = 5985  
Enabled = true  
ListeningOn = 192.168.0.183
```

### 2.6 Authentication & TrustedHosts configuration

When I retried the Ansible ping I got:

``` bash
WinRM error: The WinRM client cannot process the request. Default authentication may be used with an IP address...
```

**Cause:** The control node (client) did not trust the target IP for WinRM.
**Fix:** On the Windows 11 VM (or via GPO) I ran:

```powershell
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "*" -Force
```

# **(Note: this is acceptable only for testing; for production it should be more restrictive.)**

Next I checked the WinRM service authentication settings:

```powershell
winrm get winrm/config/service/auth  
winrm get winrm/config/service
```

I found both `Basic` and `AllowUnencrypted` were set to **false**. Since I was using basic over HTTP for now, I enabled them:

```powershell
winrm set winrm/config/service/auth '@{Basic="true"}'
winrm set winrm/config/service '@{AllowUnencrypted="true"}'
```

After this, I re-ran the Ansible ping and it still failed.

### 2.7 Securing with HTTPS and switching auth method

I then tried to configure WinRM using HTTPS (port 5986) and stronger auth. Steps:

On Windows 11 VM:

```powershell
winrm quickconfig -q

$cert = New-SelfSignedCertificate -DnsName "your_windows_host_dns_name" -CertStoreLocation Cert:\LocalMachine\My
# note the thumbprint
winrm create winrm/config/Listener?Address=*+Transport=HTTPS @{ Hostname="your_windows_host_dns_name"; CertificateThumbprint="YOUR_CERT_THUMBPRINT" }
New-NetFirewallRule -DisplayName "WinRM HTTPS" -Direction Inbound -LocalPort 5986 -Protocol TCP -Action Allow
```

On the Ansible control node:

```bash
pip install pywinrm requests-credssp
```

---
