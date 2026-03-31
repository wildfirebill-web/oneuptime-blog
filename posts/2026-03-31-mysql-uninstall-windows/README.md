# How to Uninstall MySQL Completely on Windows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Uninstall, Window, Administration, Cleanup

Description: Completely remove MySQL from Windows 10 or 11 by stopping the service, uninstalling via MySQL Installer or Programs and Features, and deleting residual data and registry entries.

---

## How It Works

On Windows, MySQL can be uninstalled through the MySQL Installer (which tracks all installed MySQL products), via Programs and Features, or with PowerShell. After uninstalling the packages, residual data directories and registry entries must be removed manually for a completely clean state.

```mermaid
flowchart LR
    A[Stop MySQL service] --> B[Uninstall via MySQL Installer]
    B --> C[Remove via Programs and Features]
    C --> D[Delete C:\ProgramData\MySQL]
    D --> E[Delete C:\Program Files\MySQL]
    E --> F[Remove registry entries]
    F --> G[Verify removal]
```

## Step 1 - Back Up Your Databases

Before uninstalling, export any databases you want to keep.

```bash
mysqldump -u root -p --all-databases > C:\Backup\all-databases.sql
```

## Step 2 - Stop the MySQL Service

Open PowerShell as Administrator.

```bash
Stop-Service -Name MySQL80 -Force
Set-Service -Name MySQL80 -StartupType Disabled
```

Or from the Services console (`Win+R`, type `services.msc`), right-click **MySQL80** and click **Stop**.

## Step 3 - Uninstall via MySQL Installer (Recommended)

If MySQL was installed using the MySQL Installer:

1. Open the **MySQL Installer** from the Start Menu.
2. The installer lists all installed MySQL products.
3. Click the **X** (Remove) icon next to **MySQL Server 8.0**.
4. Repeat for MySQL Workbench, MySQL Shell, MySQL Router, and connectors.
5. Click **Execute** to remove selected products.

## Step 4 - Uninstall via Programs and Features

If the MySQL Installer is not available:

1. Open **Settings > Apps > Installed apps** (Windows 11) or **Control Panel > Programs > Programs and Features** (Windows 10).
2. Search for "MySQL."
3. Click **MySQL Server 8.0** and click **Uninstall**.
4. Follow the uninstall wizard.
5. Repeat for all MySQL-related entries.

## Step 5 - Remove the Windows Service

After uninstalling, verify the service is gone.

```bash
Get-Service -Name MySQL80 -ErrorAction SilentlyContinue
```

If the service still exists, remove it.

```bash
sc.exe delete MySQL80
```

Or using mysqld directly:

```bash
"C:\Program Files\MySQL\MySQL Server 8.0\bin\mysqld.exe" --remove MySQL80
```

## Step 6 - Delete Residual Files

The uninstaller may not remove the data directory or all configuration files.

Open PowerShell as Administrator.

```bash
# Delete the data and configuration directory
Remove-Item -Recurse -Force "C:\ProgramData\MySQL"

# Delete the program directory (if not removed by uninstaller)
Remove-Item -Recurse -Force "C:\Program Files\MySQL"
```

Check for any remaining MySQL files in common locations.

```bash
Get-ChildItem "C:\Program Files" -Filter "*MySQL*" -ErrorAction SilentlyContinue
Get-ChildItem "C:\ProgramData" -Filter "*MySQL*" -ErrorAction SilentlyContinue
```

## Step 7 - Remove Registry Entries

MySQL writes entries to the Windows registry that should be cleaned up.

Open **Registry Editor** (`Win+R`, type `regedit`).

Navigate to and delete these keys if they exist:

```text
HKEY_LOCAL_MACHINE\SOFTWARE\MySQL AB
HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\MySQL AB
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\MySQL80
```

Or use PowerShell.

```bash
Remove-Item -Path "HKLM:\SOFTWARE\MySQL AB" -Recurse -ErrorAction SilentlyContinue
Remove-Item -Path "HKLM:\SOFTWARE\WOW6432Node\MySQL AB" -Recurse -ErrorAction SilentlyContinue
Remove-Item -Path "HKLM:\SYSTEM\CurrentControlSet\Services\MySQL80" -Recurse -ErrorAction SilentlyContinue
```

## Step 8 - Remove PATH Environment Variable Entry

Check if the MySQL bin directory is still in the system PATH.

```bash
[System.Environment]::GetEnvironmentVariable("Path", "Machine") -split ";" | Where-Object { $_ -like "*MySQL*" }
```

If it is, remove it.

```bash
$currentPath = [System.Environment]::GetEnvironmentVariable("Path", "Machine")
$newPath = ($currentPath -split ";" | Where-Object { $_ -notlike "*MySQL*" }) -join ";"
[System.Environment]::SetEnvironmentVariable("Path", $newPath, "Machine")
```

## Step 9 - Verify Complete Removal

```bash
# No MySQL service should exist
Get-Service -Name "MySQL*" -ErrorAction SilentlyContinue

# No MySQL processes should be running
Get-Process -Name "mysqld" -ErrorAction SilentlyContinue

# Port 3306 should not be listening
Get-NetTCPConnection -LocalPort 3306 -ErrorAction SilentlyContinue
```

All three commands should return no results.

## Summary

Completely removing MySQL from Windows requires stopping the service, uninstalling via MySQL Installer or Programs and Features, deleting residual data in `C:\ProgramData\MySQL` and `C:\Program Files\MySQL`, removing the Windows service with `sc.exe delete`, and cleaning up registry entries under `HKLM:\SOFTWARE\MySQL AB`. The MySQL Installer is the cleanest removal path because it tracks all installed MySQL products and removes them together.
