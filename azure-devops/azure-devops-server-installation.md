# Azure DevOps Server Installation Guide

![Azure DevOps Server](https://imgur.com/D1OFqlv.png)

Comprehensive guide for installing and configuring Azure DevOps Server (formerly Team Foundation Server) on-premises.

## Table of Contents
1. [Introduction](#1-introduction)
2. [System Requirements](#2-system-requirements)
3. [Pre-Installation Planning](#3-pre-installation-planning)
4. [Installation Steps](#4-installation-steps)
5. [Post-Installation Configuration](#5-post-installation-configuration)
6. [High Availability Setup](#6-high-availability-setup)
7. [Upgrade and Migration](#7-upgrade-and-migration)
8. [Backup and Restore](#8-backup-and-restore)
9. [Troubleshooting](#9-troubleshooting)

---

## 1. Introduction

### 1.1. What is Azure DevOps Server?

Azure DevOps Server is the on-premises version of Azure DevOps Services, providing:
- Complete control over data and infrastructure
- Compliance with data residency requirements
- Integration with on-premises systems
- Offline capabilities

### 1.2. Azure DevOps Server vs Azure DevOps Services

| Feature | Azure DevOps Server | Azure DevOps Services |
|---------|---------------------|----------------------|
| Hosting | On-premises | Cloud-hosted by Microsoft |
| Maintenance | Self-managed | Managed by Microsoft |
| Updates | Manual | Automatic |
| Cost | License required | Free for up to 5 users |
| Integration | On-premises systems | Azure services |
| Data Residency | On-premises | Microsoft datacenters |

### 1.3. Editions

- **Express**: Free for up to 5 users (single server)
- **Professional**: For teams requiring advanced features
- **Enterprise**: For large organizations with complex needs

---

## 2. System Requirements

### 2.1. Hardware Requirements

#### Minimum Requirements (Small Teams)

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| CPU | 4-core 64-bit | 8-core 64-bit |
| RAM | 8 GB | 16 GB |
| Disk (Application Tier) | 100 GB | 200 GB SSD |
| Disk (Data Tier) | 500 GB | 1 TB SSD |
| Network | 1 Gbps | 1 Gbps |

#### Recommended Requirements (Large Organizations)

| Component | Recommended |
|-----------|-------------|
| CPU | 16-core 64-bit |
| RAM | 32 GB |
| Disk (Application Tier) | 500 GB SSD |
| Disk (Data Tier) | 2 TB SSD |
| Network | 10 Gbps |

### 2.2. Software Requirements

#### Operating System

**Supported Windows Server Versions:**
- Windows Server 2022 (Standard/Datacenter)
- Windows Server 2019 (Standard/Datacenter)
- Windows Server 2016 (Standard/Datacenter)

**Note:** Windows Server Core is not supported.

#### SQL Server

**Supported SQL Server Versions:**
- SQL Server 2022 (Express/Standard/Enterprise)
- SQL Server 2019 (Express/Standard/Enterprise)
- SQL Server 2017 (Express/Standard/Enterprise)

**SQL Server Express Limitations:**
- Maximum database size: 10 GB
- Limited CPU and memory
- No high availability features

#### Browser Requirements

- Microsoft Edge (latest version)
- Google Chrome (latest version)
- Mozilla Firefox (latest version)

#### Additional Software

- .NET Framework 4.8 or later
- Windows PowerShell 5.1 or later
- IIS 10.0 or later

---

## 3. Pre-Installation Planning

### 3.1. Architecture Planning

#### Single-Server Deployment

```
┌─────────────────────────────────┐
│  Azure DevOps Server (Single)   │
│  ┌───────────────────────────┐  │
│  │ Application Tier          │  │
│  │ - Web Services            │  │
│  │ - API Services            │  │
│  │ - Background Services     │  │
│  └───────────────────────────┘  │
│  ┌───────────────────────────┐  │
│  │ Data Tier                 │  │
│  │ - SQL Server              │  │
│  │ - Reporting Services      │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

**Use Case:** Small teams (< 50 users), development/testing

#### Multi-Server Deployment

```
┌─────────────────────┐
│   Load Balancer     │
└──────────┬──────────┘
           │
    ┌──────┴──────┐
    │             │
┌───▼────┐   ┌───▼────┐
│ App 1  │   │ App 2  │
└────────┘   └────────┘
    │             │
    └──────┬──────┘
           │
    ┌──────▼────────┐
    │  SQL Server   │
    │  (Always On)  │
    └───────────────┘
```

**Use Case:** Large organizations, production environments

### 3.2. Network Planning

#### Required Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 80 | HTTP | Web access (non-SSL) |
| 443 | HTTPS | Web access (SSL) |
| 1433 | TCP | SQL Server |
| 1434 | UDP | SQL Server Browser |
| 9192 | TCP | Build Controller |
| 9191 | TCP | Build Agent |

#### DNS Configuration

```powershell
# Create DNS A record for Azure DevOps Server
Add-DnsServerResourceRecordA -Name "tfs" -ZoneName "yourdomain.com" -IPv4Address "192.168.1.10"

# Create CNAME record for friendly URL
Add-DnsServerResourceRecordCName -Name "devops" -HostNameAlias "tfs.yourdomain.com" -ZoneName "yourdomain.com"
```

### 3.3. Service Account Planning

#### Required Service Accounts

| Service | Account Type | Permissions |
|---------|--------------|-------------|
| Azure DevOps Server | Domain User | Local Administrator |
| SQL Server | Domain User | Local Administrator, SQL sysadmin |
| Build Service | Domain User | Local Administrator |
| Reporting Services | Domain User | Local Administrator |

#### Create Service Accounts

```powershell
# Create service account for Azure DevOps Server
New-ADUser -Name "TFS_Service" `
  -SamAccountName "tfs_service" `
  -UserPrincipalName "tfs_service@yourdomain.com" `
  -Path "OU=Service Accounts,DC=yourdomain,DC=com" `
  -AccountPassword (ConvertTo-SecureString "SecurePassword123!" -AsPlainText -Force) `
  -Enabled $true `
  -PasswordNeverExpires $true

# Add to local administrators group
Add-LocalGroupMember -Group "Administrators" -Member "YOURDOMAIN\tfs_service"

# Create SQL Server service account
New-ADUser -Name "SQL_Service" `
  -SamAccountName "sql_service" `
  -UserPrincipalName "sql_service@yourdomain.com" `
  -Path "OU=Service Accounts,DC=yourdomain,DC=com" `
  -AccountPassword (ConvertTo-SecureString "SecurePassword123!" -AsPlainText -Force) `
  -Enabled $true `
  -PasswordNeverExpires $true
```

---

## 4. Installation Steps

### 4.1. Install SQL Server

#### Step 1: Download SQL Server

```powershell
# Download SQL Server 2022 Express
# From: https://www.microsoft.com/en-us/sql-server/sql-server-downloads

# Or use PowerShell to download
Invoke-WebRequest -Uri "https://go.microsoft.com/fwlink/?linkid=866658" -OutFile "SQLServer2022-x64-ENU.exe"
```

#### Step 2: Install SQL Server

```powershell
# Extract SQL Server
.\SQLServer2022-x64-ENU.exe /Q /x:SQLSetup

# Install SQL Server with configuration file
.\SQLSetup\setup.exe /ConfigurationFile=SQLConfig.ini
```

**SQLConfig.ini:**
```ini
;SQL Server 2022 Configuration File
[OPTIONS]
ACTION="Install"
IACCEPTSQLSERVERLICENSETERMS="True"
QUIET="True"
QUIETSIMPLE="False"
INSTANCENAME="MSSQLSERVER"
INSTANCEID="MSSQLSERVER"
SQLCOLLATION="SQL_Latin1_General_CP1_CI_AS"
AGTSVCACCOUNT="YOURDOMAIN\sql_service"
AGTSVCPASSWORD="SecurePassword123!"
SQLSVCACCOUNT="YOURDOMAIN\sql_service"
SQLSVCPASSWORD="SecurePassword123!"
ISSVCACCOUNT="NT AUTHORITY\Network Service"
SQLSYSADMINACCOUNTS="YOURDOMAIN\tfs_service","BUILTIN\Administrators"
SECURITYMODE="SQL"
SAPWD="SecurePassword123!"
TCPENABLED="1"
NPENABLED="0"
BROWSERSVCSTARTUPTYPE="Automatic"
```

#### Step 3: Configure SQL Server

```powershell
# Enable TCP/IP protocol
Import-Module SqlServer
$smo = 'Microsoft.SqlServer.Management.Smo.Wmi.ManagedComputer'
$wmi = New-Object ($smo) $env:COMPUTERNAME
$uri = "ManagedComputer[@Name='$env:COMPUTERNAME']/ServerInstance[@Name='MSSQLSERVER']/ServerProtocol[@Name='Tcp']"
$Tcp = $wmi.GetSmoObject($uri)
$Tcp.IsEnabled = $true
$Tcp.Alter()

# Restart SQL Server service
Restart-Service -Name "MSSQLSERVER" -Force

# Configure firewall rules
New-NetFirewallRule -DisplayName "SQL Server" -Direction Inbound -Protocol TCP -LocalPort 1433 -Action Allow
New-NetFirewallRule -DisplayName "SQL Server Browser" -Direction Inbound -Protocol UDP -LocalPort 1434 -Action Allow
```

### 4.2. Install Azure DevOps Server

#### Step 1: Download Azure DevOps Server

```powershell
# Download Azure DevOps Server 2022
# From: https://visualstudio.microsoft.com/downloads/

# Or use PowerShell
Invoke-WebRequest -Uri "https://go.microsoft.com/fwlink/?linkid=2208958" -OutFile "AzureDevOpsServer2022.exe"
```

#### Step 2: Prerequisites Check

```powershell
# Run prerequisites checker
.\AzureDevOpsServer2022.exe /layout
.\AzureDevOpsServer2022.exe /prereq
```

#### Step 3: Install Azure DevOps Server

**Option 1: Basic Installation (Single Server)**

```powershell
# Run installer
.\AzureDevOpsServer2022.exe

# Follow the wizard:
# 1. Click "Install Azure DevOps Server"
# 2. Accept license terms
# 3. Choose "Basic" installation type
# 4. Configure:
#    - Application Tier URL: https://devops.yourdomain.com
#    - Database: Local SQL Server
#    - Authentication: NTLM or Kerberos
#    - Search: Install search service (optional)
# 5. Click "Install"
# 6. Wait for installation to complete
```

**Option 2: Advanced Installation (Multi-Server)**

```powershell
# Run installer
.\AzureDevOpsServer2022.exe

# Follow the wizard:
# 1. Click "Install Azure DevOps Server"
# 2. Accept license terms
# 3. Choose "Advanced" installation type
# 4. Select components:
#    - Application Tier
#    - Database (on separate server)
#    - Reporting Services (optional)
#    - Search Service (optional)
# 5. Configure each component separately
# 6. Click "Install"
```

#### Step 4: Configure Application Tier

```powershell
# Open Azure DevOps Server Administration Console
# Navigate to: Start > Azure DevOps Server Administration Console

# Configure application tier:
# 1. Click "Application Tier"
# 2. Configure:
#    - URL: https://devops.yourdomain.com
#    - Port: 443 (HTTPS)
#    - Authentication: Negotiate (Kerberos)
#    - Service Account: YOURDOMAIN\tfs_service
# 3. Click "Apply"
```

### 4.3. Configure SSL/TLS

#### Step 1: Request SSL Certificate

```powershell
# Request certificate from internal CA
$certDistinguishedName = "CN=devops.yourdomain.com, OU=IT, O=YourCompany, L=YourCity, S=YourState, C=US"
$certTemplate = "WebServer"

Get-Certificate -Template $certTemplate -DnsName "devops.yourdomain.com" -SubjectName $certDistinguishedName -CertStoreLocation "Cert:\LocalMachine\My"
```

#### Step 2: Import SSL Certificate

```powershell
# Import PFX certificate
$certPath = "C:\Certificates\devops.yourdomain.com.pfx"
$certPassword = ConvertTo-SecureString "CertificatePassword" -AsPlainText -Force
Import-PfxCertificate -FilePath $certPath -CertStoreLocation "Cert:\LocalMachine\My" -Password $certPassword
```

#### Step 3: Bind SSL Certificate

```powershell
# Bind certificate to IIS
$certThumbprint = (Get-ChildItem -Path "Cert:\LocalMachine\My" | Where-Object {$_.Subject -like "*devops.yourdomain.com*"}).Thumbprint

# Configure IIS binding
Import-Module WebAdministration
New-WebBinding -Name "Default Web Site" -Protocol https -Port 443 -HostHeader "devops.yourdomain.com"

# Bind certificate
$binding = Get-WebBinding -Name "Default Web Site" -Protocol https
$binding.AddSslCertificate($certThumbprint, "My")
```

### 4.4. Create Project Collection

#### Step 1: Create Default Collection

```powershell
# Open Azure DevOps Server Administration Console
# Navigate to: Project Collections

# Click "Create Collection"
# Configure:
# - Collection Name: DefaultCollection
# - Description: Default project collection
# - Database: Create new database
# - Database Name: TFS_DefaultCollection
# - Reporting Services: Configure if needed
# Click "Create"
```

#### Step 2: Configure Collection Settings

```powershell
# Configure collection properties
# Navigate to: Project Collections > DefaultCollection > General

# Configure:
# - Collection URL: https://devops.yourdomain.com/DefaultCollection
# - Service Account: YOURDOMAIN\tfs_service
# - Permission Model: Project Collection
# Click "Save"
```

---

## 5. Post-Installation Configuration

### 5.1. Configure Build and Release Agents

#### Step 1: Create Agent Pool

```powershell
# Open Azure DevOps Server web portal
# Navigate to: Project Settings > Agent Pools

# Click "Add pool"
# Configure:
# - Pool Name: Default
# - Type: Self-hosted
# Click "Create"
```

#### Step 2: Install Build Agent

```powershell
# Create agent directory
mkdir C:\Agent
cd C:\Agent

# Download agent package
# Navigate to: https://devops.yourdomain.com/_admin/_AgentPool
# Download Windows agent package

# Extract agent
Add-Type -AssemblyName System.IO.Compression.FileSystem
[System.IO.Compression.ZipFile]::ExtractToDirectory("agent.zip", ".")

# Configure agent
.\config.cmd

# Follow prompts:
# - Enter server URL: https://devops.yourdomain.com
# - Enter authentication type: Integrated
# - Enter agent pool: Default
# - Enter agent name: (accept default)
# - Enter work folder: _work
# - Run as service: Y
# - Service account: YOURDOMAIN\tfs_service

# Start agent
.\run.cmd
```

### 5.2. Configure Search Service

#### Step 1: Install Search Service

```powershell
# Open Azure DevOps Server Administration Console
# Navigate to: Application Tier > Search

# Click "Install Search Service"
# Configure:
# - Search Service Account: YOURDOMAIN\tfs_service
# - Search Service URL: https://devops.yourdomain.com
# Click "Install"
```

#### Step 2: Configure Search Indexes

```powershell
# Configure search settings
# Navigate to: Application Tier > Search > Indexes

# Configure:
# - Code Search: Enable
# - Work Item Search: Enable
# - Wiki Search: Enable
# Click "Save"
```

### 5.3. Configure Reporting Services

#### Step 1: Install Reporting Services

```powershell
# Install SQL Server Reporting Services
# Download from: https://www.microsoft.com/en-us/sql-server/sql-server-downloads

# Run installer
.\SSRSSetup.exe

# Follow wizard:
# 1. Install Reporting Services
# 2. Configure Report Server
# 3. Connect to Azure DevOps Server
```

#### Step 2: Configure Reports

```powershell
# Open Azure DevOps Server Administration Console
# Navigate to: Project Collections > DefaultCollection > Reporting

# Configure:
# - Report Server URL: https://reports.yourdomain.com/ReportServer
# - Report Manager URL: https://reports.yourdomain.com/Reports
# Click "Apply"
```

### 5.4. Configure Email Notifications

#### Step 1: Configure SMTP Server

```powershell
# Open Azure DevOps Server Administration Console
# Navigate to: Application Tier > Email Settings

# Configure:
# - SMTP Server: smtp.yourdomain.com
# - SMTP Port: 587
# - From Address: devops@yourdomain.com
# - Use SSL: Yes
# - SMTP User: devops@yourdomain.com
# - SMTP Password: [password]
# Click "Test Email"
# Click "Save"
```

---

## 6. High Availability Setup

### 6.1. SQL Server Always On

#### Step 1: Configure Always On Availability Group

```powershell
# Enable Always On Availability Groups
Enable-SqlAlwaysOn -ServerName "SQL01" -Force

# Create availability group
New-SqlAvailabilityGroup `
  -Name "TFS_AG" `
  -Path "SQLSERVER:\SQL\SQL01\Default" `
  -AvailabilityReplica (
    New-SqlAvailabilityReplica `
      -Name "SQL01" `
      -EndpointUrl "TCP://SQL01.yourdomain.com:5022" `
      -FailoverMode "Automatic" `
      -AvailabilityMode "SynchronousCommit" `
      -ConnectionModeInSecondaryRole "AllowAllConnections" `
      -AsTemplate
  ),
  (New-SqlAvailabilityReplica `
      -Name "SQL02" `
      -EndpointUrl "TCP://SQL02.yourdomain.com:5022" `
      -FailoverMode "Automatic" `
      -AvailabilityMode "SynchronousCommit" `
      -ConnectionModeInSecondaryRole "AllowAllConnections" `
      -AsTemplate
  )

# Add database to availability group
Add-SqlAvailabilityDatabase `
  -Path "SQLSERVER:\SQL\SQL01\Default\AvailabilityGroups\TFS_AG" `
  -Database "TFS_Configuration"
```

#### Step 2: Configure Listener

```powershell
# Create availability group listener
$sqlAG = Get-SqlAvailabilityGroup -Path "SQLSERVER:\SQL\SQL01\Default\AvailabilityGroups\TFS_AG"
New-SqlAvailabilityGroupListener `
  -Name "TFS_Listener" `
  -StaticIp "192.168.1.100/255.255.255.0" `
  -Port 1433 `
  -Path $sqlAG.Path
```

### 6.2. Application Tier High Availability

#### Step 1: Configure Load Balancer

```powershell
# Install Network Load Balancing (NLB) on Windows Server
# Or use hardware load balancer

# Configure NLB cluster
Import-Module NetworkLoadBalancingClusters
New-NlbCluster -ClusterName "TFS_NLB" -ClusterPrimaryIP "192.168.1.200" -SubnetMask "255.255.255.0"
Add-NlbClusterNode -ClusterName "TFS_NLB" -HostName "TFS01"
Add-NlbClusterNode -ClusterName "TFS_NLB" -HostName "TFS02"

# Configure port rules
Add-NlbClusterPortRule -ClusterName "TFS_NLB" -StartPort 80 -EndPort 80 -Protocol TCP -Mode Multiple
Add-NlbClusterPortRule -ClusterName "TFS_NLB" -StartPort 443 -EndPort 443 -Protocol TCP -Mode Multiple
```

#### Step 2: Configure Application Tier

```powershell
# On each application tier server:
# Install Azure DevOps Server (Application Tier only)
# Configure to use same SQL Server availability group
# Configure with same URL and settings
```

---

## 7. Upgrade and Migration

### 7.1. Upgrade Azure DevOps Server

#### Step 1: Pre-Upgrade Checklist

```powershell
# Backup databases
# Backup encryption keys
# Check system requirements
# Review release notes
# Test upgrade in staging environment
```

#### Step 2: Perform Upgrade

```powershell
# Download latest Azure DevOps Server version
# From: https://visualstudio.microsoft.com/downloads/

# Run upgrade wizard
.\AzureDevOpsServer2022.exe

# Follow wizard:
# 1. Click "Upgrade Azure DevOps Server"
# 2. Accept license terms
# 3. Review upgrade summary
# 4. Click "Upgrade"
# 5. Wait for upgrade to complete
# 6. Verify upgrade success
```

### 7.2. Migrate from TFS to Azure DevOps Server

#### Step 1: Prepare for Migration

```powershell
# Install TFS Migration Tools
# Download from: https://github.com/microsoft/tfsmigrationtools

# Configure migration
# Create configuration.json file
```

**configuration.json:**
```json
{
  "Source": {
    "Collection": "https://oldtfs.yourdomain.com/DefaultCollection",
    "Project": "OldProject",
    "ReflectedWorkItemIDFieldName": "TfsMigrationTool.ReflectedWorkItemId"
  },
  "Target": {
    "Collection": "https://devops.yourdomain.com/DefaultCollection",
    "Project": "NewProject",
    "ReflectedWorkItemIDFieldName": "TfsMigrationTool.ReflectedWorkItemId"
  },
  "WorkItems": {
    "Enabled": true
  }
}
```

#### Step 2: Run Migration

```powershell
# Run migration tool
.\TfsMigrationTool.exe configuration.json

# Monitor migration progress
# Verify migration results
```

---

## 8. Backup and Restore

### 8.1. Backup Configuration

#### Step 1: Backup Encryption Key

```powershell
# Open Azure DevOps Server Administration Console
# Navigate to: Application Tier > Backup

# Click "Backup Encryption Key"
# Save to secure location
```

#### Step 2: Configure Backup Schedule

```powershell
# Open Azure DevOps Server Administration Console
# Navigate to: Application Tier > Scheduled Backup

# Configure:
# - Backup Location: \\backup-server\AzureDevOps
# - Schedule: Daily at 2:00 AM
# - Retention: 30 days
# Click "Save"
```

### 8.2. Manual Backup

#### Step 1: Backup Databases

```powershell
# Backup all Azure DevOps databases
$databases = @("TFS_Configuration", "TFS_DefaultCollection", "TFS_Warehouse", "ReportServer", "ReportServerTempDB")

foreach ($db in $databases) {
    $backupFile = "C:\Backups\$db_$(Get-Date -Format 'yyyyMMdd').bak"
    Backup-SqlDatabase -ServerInstance "localhost" -Database $db -BackupFile $backupFile
    Write-Host "Backed up $db to $backupFile"
}
```

#### Step 2: Backup File System

```powershell
# Backup Azure DevOps Server configuration
$backupPath = "C:\Backups\FileSystem_$(Get-Date -Format 'yyyyMMdd')"
Copy-Item "C:\Program Files\Azure DevOps Server" -Destination "$backupPath\AzureDevOpsServer" -Recurse
Copy-Item "C:\Program Files\Microsoft Team Foundation Server" -Destination "$backupPath\TFS" -Recurse
```

### 8.3. Restore Procedure

#### Step 1: Restore Databases

```powershell
# Stop Azure DevOps Server services
Stop-Service -Name "TFSJobAgent" -Force
Stop-Service -Name "IISADMIN" -Force

# Restore databases
Restore-SqlDatabase -ServerInstance "localhost" -Database "TFS_Configuration" -BackupFile "C:\Backups\TFS_Configuration_20240101.bak" -ReplaceDatabase
Restore-SqlDatabase -ServerInstance "localhost" -Database "TFS_DefaultCollection" -BackupFile "C:\Backups\TFS_DefaultCollection_20240101.bak" -ReplaceDatabase

# Start services
Start-Service -Name "IISADMIN"
Start-Service -Name "TFSJobAgent"
```

#### Step 2: Restore Encryption Key

```powershell
# Open Azure DevOps Server Administration Console
# Navigate to: Application Tier > Backup

# Click "Restore Encryption Key"
# Select backup file
# Enter password (if encrypted)
# Click "Restore"
```

---

## 9. Troubleshooting

### 9.1. Common Installation Issues

#### Issue: SQL Server Connection Failed

**Solution:**
```powershell
# Check SQL Server service status
Get-Service -Name "MSSQLSERVER"

# Check SQL Server configuration
Get-SqlServerNetworkConfiguration -ServerInstance "localhost"

# Test SQL connection
Test-NetConnection -ComputerName localhost -Port 1433

# Verify SQL authentication
sqlcmd -S localhost -U sa -P password
```

#### Issue: Application Tier Not Starting

**Solution:**
```powershell
# Check IIS status
Get-Service -Name "IISADMIN"
Get-Service -Name "W3SVC"

# Check application pool
Import-Module WebAdministration
Get-WebAppPoolState -Name "TFSAppPool"

# Restart IIS
iisreset

# Check event logs
Get-EventLog -LogName Application -Source "TFS Job Agent" -Newest 50
```

#### Issue: Agent Cannot Connect

**Solution:**
```powershell
# Check agent service
Get-Service -Name "VSOAgent*"

# Test network connectivity
Test-NetConnection -ComputerName devops.yourdomain.com -Port 443

# Reconfigure agent
cd C:\Agent
.\config.cmd remove
.\config.cmd
```

### 9.2. Performance Issues

#### Issue: Slow Performance

**Solution:**
```powershell
# Check database size
sqlcmd -S localhost -Q "SELECT name, size/128.0 AS SizeInMB FROM sys.master_files WHERE database_id > 4"

# Check database fragmentation
sqlcmd -S localhost -Q "SELECT * FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED')"

# Rebuild indexes
sqlcmd -S localhost -Q "ALTER INDEX ALL ON [TFS_Configuration].[tbl_AccessControlEntry] REBUILD"

# Update statistics
sqlcmd -S localhost -Q "UPDATE STATISTICS [TFS_Configuration].[tbl_AccessControlEntry]"
```

#### Issue: High Memory Usage

**Solution:**
```powershell
# Configure SQL Server memory limits
sqlcmd -S localhost -Q "EXEC sp_configure 'max server memory', 8192; RECONFIGURE"

# Configure application pool memory limits
Set-WebConfigurationProperty -Filter "system.applicationHost/applicationPools/add[@name='TFSAppPool']" -Name "processModel.privateMemoryMemoryLimit" -Value 1048576
```

---

## 10. Maintenance

### 10.1. Regular Maintenance Tasks

#### Daily Tasks

- Monitor backup status
- Check service health
- Review error logs

#### Weekly Tasks

- Review disk space
- Check database growth
- Update statistics

#### Monthly Tasks

- Review security logs
- Update SSL certificates
- Test restore procedures

### 10.2. Monitoring

#### Configure Monitoring

```powershell
# Create monitoring script
$services = @("TFSJobAgent", "IISADMIN", "W3SVC", "MSSQLSERVER")

foreach ($service in $services) {
    $status = Get-Service -Name $service -ErrorAction SilentlyContinue
    if ($status.Status -ne "Running") {
        Write-Host "Service $service is not running" -ForegroundColor Red
        # Send alert
    }
}
```

---

## References

- [Azure DevOps Server Documentation](https://docs.microsoft.com/azure/devops/server/)
- [Azure DevOps Server Installation Guide](https://docs.microsoft.com/azure/devops/server/install/overview)
- [SQL Server Documentation](https://docs.microsoft.com/sql/sql-server/)
- [High Availability Configuration](https://docs.microsoft.com/azure/devops/server/admin/high-availability)

---

## By [Harshhaa Reddy](https://www.github.com/NotHarshhaa)
