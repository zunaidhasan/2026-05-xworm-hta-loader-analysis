# 🧹 Incident Response & Cleanup Guide

This guide details the forensic checklist and remediation procedures to identify, isolate, and remove the XWorm HTA Loader and RAT from an infected Windows system.

---

## 🔍 Step 1: Host Triage & Process Hunting

Before deleting files, terminating processes in the correct order is critical to prevent the malware from self-healing or re-spawning via active persistence loops.

### **1. Identify & Terminate Malicious Processes**
Open PowerShell as Administrator and search for the following active processes:

- **MSHTA Process**:
  ```powershell
  Get-CimInstance Win32_Process -Filter "Name = 'mshta.exe'" | Select-Process
  ```
  Look for processes running with HTA files in their command line, particularly `Barge Denver Waalhaven - Work presentation.hta`.

- **WScript/CScript executing the dropped loader**:
  ```powershell
  Get-CimInstance Win32_Process -Filter "Name = 'wscript.exe' or Name = 'cscript.exe'" | Where-Object { $_.CommandLine -like "*Telegram_Private_Call_Session.vbs*" }
  ```

- **Obfuscated PowerShell Stages**:
  ```powershell
  Get-CimInstance Win32_Process -Filter "Name = 'powershell.exe'" | Where-Object { $_.CommandLine -like "*-enc*" -or $_.CommandLine -like "*lightlife*" }
  ```

### **2. Kill Active Processes**
If identified, terminate them immediately (starting with `mshta.exe` and `wscript.exe` to stop persistence coordination):
```powershell
Stop-Process -Name "mshta" -Force
Stop-Process -Name "wscript" -Force
Stop-Process -Name "powershell" -Force
```

---

## 📂 Step 2: File System Remediation

The loader leaves specific host indicators in the local user AppData folders. Navigate to `C:\Users\<Username>\AppData\Local\` and remove the following artifacts:

| Filename | SHA256 Hash | Target Path | Action |
|---|---|---|---|
| `started.flag` | `ECBC89CD37A037342E740F9E0E0633A70180A2279B0157C9A823FCE729F4BD77` | `%LocalAppData%\started.flag` | **Permanently Delete** |
| `Telegram_Private_Call_Session.vbs` | `9DF997476E96979E19BD6EDA529B62DD9F1FE350DFD2E1C84F86D5DF48CE8871` | `%LocalAppData%\Telegram_Private_Call_Session.vbs` | **Permanently Delete** |

### **PowerShell File Cleanup Command**:
```powershell
$TargetDir = "$env:USERPROFILE\AppData\Local"
$FilesToRemove = @("started.flag", "Telegram_Private_Call_Session.vbs")

foreach ($File in $FilesToRemove) {
    $FilePath = Join-Path $TargetDir $File
    if (Test-Path $FilePath) {
        Remove-Item -Path $FilePath -Force -Confirm:$false
        Write-Output "[+] Successfully removed: $FilePath"
    } else {
        Write-Output "[-] File not found: $FilePath"
    }
}
```

---

## 🔑 Step 3: Persistence & Registry Cleanup

The malware configures auto-start mechanisms to ensure it reloads upon system boot. Check and remediate the following vectors:

### **1. Scheduled Tasks**
The loader leverages Windows Scheduled Tasks to launch the dropped VBS script. Run the following command to query for suspicious tasks:
```powershell
Get-ScheduledTask | Where-Object { $_.Actions.Execute -like "*wscript*" -and $_.Actions.Arguments -like "*Telegram_Private_Call_Session.vbs*" }
```
If a matching task is found, delete it:
```powershell
Unregister-ScheduledTask -TaskName "Telegram_Private_Call_Session" -Confirm:$false
```
*(Verify task names matching `Telegram`, `Waalhaven`, `Denver`, or suspicious random strings).*

### **2. Registry Run Keys**
Check for auto-run entries in the current user hive:
```powershell
Get-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run"
```
Look for values pointing to `wscript.exe` executing `%LocalAppData%\Telegram_Private_Call_Session.vbs`. If found, delete the property:
```powershell
Remove-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" -Name "TelegramSession" -Force
```

### **3. Startup Directory**
Check the user's Startup directory for shortcut (`.lnk`) or script files:
`C:\Users\<Username>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\`
Remove any suspicious items referencing `wscript.exe`, `mshta.exe`, or the local AppData scripts.

---

## 🌐 Step 4: Network Containment & DNS Sinkholing

To prevent further beaconing or remote control capabilities:

1. **Firewall Block Rules**:
   Add a local firewall rule to block all communication to the distribution C2 IP address:
   - Target IP: `193.23.202.187`
   - Portmap IP: `193.161.193.99`

2. **Hosts File Sinkholing**:
   Add sinkhole records to the Windows hosts file (`C:\Windows\System32\drivers\etc\hosts`) to redirect traffic destined for the C2 domain to localhost:
   ```text
   127.0.0.1 dayzcheatcheck.online
   127.0.0.1 jjjjjjjujjj-55237.portmap.io
   ```

---

## 🛡️ Step 5: Post-Remediation Verification

After completing the cleanup, verify that the system is fully clean:
1. Reboot the target computer.
2. Check Task Manager / Process Explorer to confirm no unexplained `wscript.exe` or `powershell.exe` instances are running.
3. Validate that `%LocalAppData%\started.flag` and `Telegram_Private_Call_Session.vbs` have not re-appeared (which would indicate an active secondary persistence mechanism).
4. Run a full endpoint scan using updated signature-based antivirus or EDR solutions.
