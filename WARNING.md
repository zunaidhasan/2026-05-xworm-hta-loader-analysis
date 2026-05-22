# ⚠️ Malware Analysis Warning & Disclaimer

> [!CAUTION]
> **HIGH RISK MATERIAL**
> This repository contains detailed threat intelligence, forensic indicators, and descriptions of a live malware loader campaign (`XWorm HTA Loader`). The indicators, files, and scripts described herein are highly malicious. 

## 🚨 Guidelines for Safe Handling

If you are replicating this analysis, attempting dynamic extraction, or handling any associated artifacts, you **MUST** strictly adhere to the following safety protocols:

1. **Air-Gapped / Isolated Sandbox Only**
   - Under no circumstances should any HTA, VBS, or PowerShell scripts from this campaign be executed on a host system or any machine connected to a production network.
   - Use dedicated, non-persistent malware analysis virtual environments (e.g., CAPE Sandbox, Zenbox, FLARE VM) with network isolation enabled.

2. **Network Danger (Active C2 Infrastructure)**
   - The loaders in this campaign dynamically attempt network callbacks to a known German IP address: `193.23.202.187` on port `8022`.
   - The secondary stage payloads communicate with active command-and-control (C2) domains, including:
     - `dayzcheatcheck.online` (resolving via Cloudflare proxy IPs: `104.21.52.212`, `172.67.204.25`)
     - `jjjjjjjujjj-55237.portmap.io` (a port-forwarding redirection tunnel resolving to Portmap.io backend `193.161.193.99`)
   - Ensure your hypervisor's network interface is configured to **Host-Only** or **Isolated** (using tools like `INetSim` or `fakenet-ng` to simulate C2 responses) to prevent live outbound connections to these malicious endpoints.

3. **Obfuscated PowerShell Execution**
   - The HTA loader makes extensive use of heavily obfuscated, base64-encoded PowerShell payloads designed to bypass basic AMSI (Antimalware Scan Interface) rules. Be extremely careful when copy-pasting or decoding these command strings on live hosts.

## ⚖️ Legal Disclaimer

All information and analysis in this repository are provided strictly for **educational, defensive, and security research purposes**. 

The authors and contributors:
- Assume no liability or responsibility for any damage, loss of data, or system compromise caused by the misuse or improper handling of these indicators or files.
- Do not condone, facilitate, or support any unauthorized access or malicious activities.
- Provide all data "as-is" without warranty of any kind.

***
**Handle with care. Stay safe.**
