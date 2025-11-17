# GSALocalAccess Script for MacOS
The GSALocalAccess-Mac script is designed to enhance the user experience for Global Secure Access (GSA) users by seamlessly managing Private Access based on network connectivity. This script automates the enabling and disabling of Private Access without requiring any user intervention. When a user connects to the corporate network, Private Access is automatically disabled, ensuring smooth internal resource access. Conversely, upon disconnection from the corporate network, Private Access is automatically reactivated, maintaining secure remote connectivity.

# How GSALocalAccess-Mac works
The Corp Network Detector runs as a macOS LaunchAgent in the user's session and monitors for network changes every 15 seconds by checking the network signature (interface, WiFi SSID, and IP address). When it detects that the network has changed—such as switching from home WiFi to corporate WiFi, connecting/disconnecting VPN, or changing IP addresses—it triggers a script that performs a DNS lookup to resolve a corporate domain. If the domain resolves successfully, the script determines the user is on the corporate network and sets the Microsoft Global Secure Access preference IsPrivateAccessDisabledByUser to 1 (disabling private access since corporate resources are directly accessible). If the DNS lookup fails, the user is outside the corporate network, and the preference is set to 0 (enabling private access to reach corporate resources through the secure tunnel).

The script includes intelligent safeguards: it only updates the preference if the current value differs from what's expected, verifies that the change was applied successfully, and logs all actions for troubleshooting. All three log files automatically rotate when they exceed 10,000 lines to prevent disk space issues. The solution requires Full Disk Access for /bin/bash because Microsoft Global Secure Access stores its preferences in a sandboxed container that normal user processes cannot access.

### Description of files
- **corp_network_check.sh** - The main monitoring script that runs continuously, checking network signatures every 15 seconds, performing DNS lookups when networks change, and updating the Microsoft Global Secure Access preference to enable/disable private access based on corporate network presence.
- **com.intune.corpnetwork.plist** - The LaunchAgent configuration file that tells macOS to automatically start the monitoring script when the user logs in, keep it running, and restart it if it crashes or when the network becomes available.
- **install.sh** - The installation script that validates files are in place, sets correct permissions, creates the log file with proper ownership, and loads the LaunchAgent into the user's session to start monitoring.
- **uninstall.sh** - The removal script that stops and unloads the LaunchAgent, kills any running processes, and deletes all installed files and directories to completely remove the Corp Network Detector from the system.
- **status.sh** - A diagnostic utility script that checks if the agent is running, displays current network status, shows the Microsoft Global Secure Access preference value, verifies Full Disk Access permissions, and displays recent log entries for troubleshooting.
- **README.md** - (Note, this file was AI generated) Comprehensive documentation that explains how the solution works, provides installation instructions, describes configuration options, includes troubleshooting steps, and documents all technical details for administrators and users. 

## Script requirements
- Global Secure Access Client should be installed before running the script.
- Intune license, if you need to to push the script using Microsoft Intune.
- An app or other method to create a `.pkg` file such as Packages, pkgbuild, Composer, etc.

# Prepare the .sh script
Whether you are installing this manually or via Intune, you first need to update the `.sh` script. Perform the following steps:
1. Download the corp-network-detector-mac.zip file to your Mac.
2. Extract or unzip the file.
3. Open and modify the corp_network_check.sh file:

| Value to modify | Description | Can leave default value? | 
|-------------|------------------|------------------|
| CORP_DOMAIN | Set this to a FQDN that can only be DNS-resolved when connected to the corp network. | ❌No - Must update |
| LOG_FILE | File path where script logs are stored. Default path is "/Users/Shared/corp_network_check.log". Set this to a path where you want logs to be stored. | ✅Yes (update optional) |
| STATE_FILE | This file is used to remember the last known network state (on vs off corpnet) which is used as part of the script logic. Default path is "/tmp/corp_network_last_state". You may change this path if you want. | ✅Yes (update optional) |
| CHECK_INTERVAL | Number of seconds between checks. Default is 15 seconds. A check will determine if a network change has occurred. Only if a network change is detected, will the subsequent DNS check occur.  | ✅Yes (update optional) |
| MAX_LOG_LINES | Maximum number of log entries to keep before older logs begin being overwritten. Default is 10000. Main monitoring logs will be rotated instantly once this threshold is reached while the Standard Output and Standard Error logs will be rotated less less freqently (default every 1 hour).  | ✅Yes (update optional) |
| ROTATION_COUNTER | Starting counter for periodic log rotation. Default is 0.  | ✅Recommend leave at default |
| ROTATION_INTERVAL | Number of checks (from CHECK_INTERVAL) before rotating the Stdout/Stderr Logs if they exceed the MAX_LOG_LINES value. Default is 240. 240 x 15 seconds = 1 hour. | ✅Recommend leave at default |

4. Save the changes.

# Deploy the script manually
Open Terminal and run the following commands:
```
# Navigate to the unzipped directory
cd /path/to/corp-network-detector

# Copy the plist to LaunchAgents
sudo cp com.intune.corpnetwork.plist /Library/LaunchAgents/

# Create the script directory
sudo mkdir -p "/Library/Application Support/corp-network-detector"

# Copy the monitoring script
sudo cp corp_network_check.sh "/Library/Application Support/corp-network-detector/"

# Make the script executable
sudo chmod +x "/Library/Application Support/corp-network-detector/corp_network_check.sh"

# (Optional) Copy status script for troubleshooting
sudo cp status.sh "/Library/Application Support/corp-network-detector/"
sudo chmod +x "/Library/Application Support/corp-network-detector/status.sh"

# Run the install script to load and start the agent
sudo bash install.sh

# Verify it's running
tail -f /Users/Shared/corp_network_check.log
```

# Uninstall the script manually
Open Terminal and run the following commands:

   ```
# Navigate to the unzipped directory
cd /path/to/corp-network-detector

# Run the uninstall script
sudo bash ./uninstall.sh
```

# Deploy the script via Intune
Before you begin creating the `.pkg` file be sure you have modified the **corp_network_check.sh** file as needed.

## Step 1: Create the .pkg file (Generic Instructions)
This guide provides generic instructions for creating a macOS installer package (.pkg) for the Corp Network Detector. These instructions should work with any package creation tool (Packages, pkgbuild, Composer, etc.).

---

### Package Overview

**Package Name**: Corp Network Detector  
**Package Identifier**: `com.intune.corpnetwork`  
**Version**: 2.0  
**Install Location**: `/` (root)  
**Requires Admin**: Yes

---

### Files to Include

#### Payload Files

The package must install exactly **2 required files** and **1 optional file**:

| Source File | Destination Path | Owner | Group | Permissions | Required |
|-------------|------------------|-------|-------|-------------|----------|
| `com.intune.corpnetwork.plist` | `/Library/LaunchAgents/com.intune.corpnetwork.plist` | root | wheel | 644 (rw-r--r--) | ✅ Required |
| `corp_network_check.sh` | `/Library/Application Support/corp-network-detector/corp_network_check.sh` | root | wheel | 755 (rwxr-xr-x) | ✅ Required |
| `status.sh` | `/Library/Application Support/corp-network-detector/status.sh` | root | wheel | 755 (rwxr-xr-x) | ⚪ Optional |

#### Directory Structure

The package payload must create this directory structure:
```
/
├── Library/
│   ├── LaunchAgents/
│   │   └── com.intune.corpnetwork.plist
│   └── Application Support/
│       └── corp-network-detector/
│           ├── corp_network_check.sh
│           └── status.sh (optional)
```

---

### Postinstall Script

The package **must** include a postinstall script that runs **after** the files are installed.

**Script Content:**

Copy the entire contents of the `install.sh` file into your postinstall script.

**Script Requirements:**
- Must run as **root**
- Must be executable (755 permissions)

---

### Package Settings

Configure your package with these settings:

| Setting | Value |
|---------|-------|
| **Package Identifier** | `com.intune.corpnetwork` |
| **Package Version** | `2.0` |
| **Install Location** | `/` |
| **Requires Admin Authentication** | Yes (checked) |
| **Allow Relocation** | No |
| **Allow Downgrade** | No (recommended) |
| **Minimum OS Version** | macOS 10.14 (Mojave) or later |

---

## (alternative) Step 1: Create the .pkg file using Packages
1. Download and install the [Packages](https://packages.macupdate.com) application on your Mac.
2. Open **Packages**.
3. Select **Distribution** and click **Next**.
4. Enter a Project Name and Project Directory. Click **Create**.
5. Select the Package you just created on the left nav bar.

![alt text](https://github.com/JeffBley/GSALocalAccess-Mac/blob/main/images/Image1.png?raw=true)

7. In the **Settings** tab, enter an identifier value for this package (example: com.yourcompanyname.appname). Leave the other fields at their default values.

![alt text](https://github.com/JeffBley/GSALocalAccess-Mac/blob/main/images/Image2.png?raw=true)

8. Select the **Payload** tab.
9. Expand the **Library** folder and right-click on the **Application Support** folder. Select **New Folder**.
10. Name the folder **corp-network-detector**. NOTE! The folder must be called exactly as shown (case-sensitive).
11. Right-click the folder **corp-network-detector** and select **Add Files**.
12. Navigate to and select the **corp_network_check.sh** file. Click **Add**.
13. Next, right-click on the folder **LaunchAgents** (located under Library) and click **Add Files**.
14. Navigate to and select the **com.intune.corpnetwork.plist** file. Click **Add**.

![alt text](https://github.com/JeffBley/GSALocalAccess-Mac/blob/main/images/Image3.png?raw=true)

15.  Select the **Scripts** tab.
16.  Under **Post-installation** click **Choose**.
17.  Navigate to and select the **install.sh** file. Click **Choose**.
18.  In the menu bar, click **Build** and select **Build**.

![alt text](https://github.com/JeffBley/GSALocalAccess-Mac/blob/main/images/Image4.png?raw=true)

19.   Follow any prompts to save or allow the action.
20.   Open the Build folder to find your `.pkg` file (saved in the Project Directory you configured in step #4).  

## Step 2: Deploy the .pkg file with Intune
1. Go to Intune.microsoft.com > Apps > macOS > Click **Create**. 
2. Under App Type select **macOS app (PKG)**
3. Click **Select**. 
4. Select the `.pkg` file and click **OK**. 
5. Give it a name, description, and publisher. Click **Next**. 
6. Click **Next** (Skipping Program).
7. Select a minimum operating system and click **Next**. 
8. Click **Next** (leaving detection rules at default).
9. Assign to your user(s). Click **Next**. 
10. Click **Create**.

# Rollback script post-Intune Deployment
To uninstall the `.pkg` file you deployed via Intune, you must unassign your users from the GSALocalAccess-Mac app in Intune and create and deploy an uninstall `.pkg` file. You may either use the `UninstallGSALocalAccess-Mac.pkg` file in this GitHub repo or create your own. If you create your own, create an empty `.pkg` file and upload the uninstall.sh script as the post-installation script. Deploy the UninstallGSALocalAccess-Mac.pkg via Intune as described [above](https://github.com/JeffBley/GSALocalAccess-Mac/tree/main?tab=readme-ov-file#step-2-deploy-the-pkg-file-with-intune).

# Troubleshooting Logs and Their Purposes
## 1. Main Monitoring Log
Location: /Users/Shared/corp_network_check.log
What it contains:

- Network signature changes (WiFi switches, IP changes, interface changes)
- DNS resolution results (inside/outside corp network)
- Preference updates (when IsPrivateAccessDisabledByUser changes)
- Preference verification (confirms value was written correctly)
- Log rotation events
- Installation/uninstallation activities

Example entries:
```
Fri Oct 24 15:47:43 EDT 2025: Network change detected (en1::192.168.1.34 → en1::192.168.1.27)
Fri Oct 24 15:47:43 EDT 2025: ❌ Outside corp network (failed to resolve private.edgediagnostic.globalseecureaccess.microsoft.com)
Fri Oct 24 15:47:43 EDT 2025: ⚙️  Updated IsPrivateAccessDisabledByUser from 1 to 0
Fri Oct 24 15:47:43 EDT 2025: ✓ Verified preference value is correct: 0
```

**When to check this log**: This is your **primary log** for monitoring and troubleshooting. Check this to see if network changes are being detected and preferences are being updated correctly.

---

## 2. **Standard Output Log**
**Location**: `/Users/Shared/corp_network_check_stdout.log`

**What it contains:**
- Script startup messages (`echo` statements)
- Basic operational messages
- Any output from commands that write to stdout

**Example entries:**
```
Starting corp network check (network change monitoring mode)...
Logging to /Users/Shared/corp_network_check.log
```

**When to check this log**: Usually only contains startup messages. Check this if the script isn't starting at all or you suspect the script is crashing immediately.

---

## 3. **Standard Error Log**
**Location**: `/Users/Shared/corp_network_check_stderr.log`

**What it contains:**
- Error messages from bash
- Permission denied errors
- Command failures
- Script syntax errors
- Any output written to stderr by commands

**Example entries (when things go wrong):**
```
/Library/Application Support/corp-network-detector/corp_network_check.sh: line 35: /Users/Shared/corp_network_check.log: Permission denied
2025-10-24 12:56:26.592 defaults[15106:662085] Could not write domain /Users/jeffreybley/Library/Containers/com.microsoft.globalsecureaccess/Data/Library/Preferences/com.microsoft.globalsecureaccess; exiting
When to check this log: Check this when things aren't working - it will show you actual errors that occurred while the script was running.
