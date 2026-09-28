# Windows Privilege Escalation



Before trying to exploit a vulnerability or running any privilege escalation tool, you should first perform basic enumeration of the environment.

The goal is to understand where you are, what network you are connected to, and what security controls are protecting the system. This information helps you choose the appropriate technique instead of blindly running tools.

### Network Enumeration

#### `ipconfig /all`

Displays the network configuration of all network adapters, including IP addresses, MAC addresses, DNS servers, and default gateways.

#### `arp -a`

Displays the host's ARP cache, which maps IP addresses to MAC addresses for devices on the local network.

#### &#x20;`route print`

Displays the Windows IP routing table.

#### **Security Control Enumeration**

Before executing tools or exploits, it is also important to identify the security controls active on the system.

**`Get-MpComputerStatus`**

Displays the current status of Microsoft Defender Antivirus.

&#x20;**`Get-AppLockerPolicy -Effective`**

Displays the effective AppLocker policy applied to the current system.

**`Test-AppLockerPolicy`**

Tests whether a specific file would be allowed or blocked by the current AppLocker policy without actually executing the file.

&#x20;**`Get-ItemProperty` — Registry Enforcement Check**

Reads values from Windows Registry keys..

