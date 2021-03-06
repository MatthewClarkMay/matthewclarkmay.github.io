---
layout: post
title: CMD Cheat Sheet
tags: [incident-response, pivoting, powershell, windows]
comments: false
---
This page is a command reference for various blue and red team tactics and techniques. I've accumulated most of this content from SANS posters, various blogs, and personal experience. It is meant to be a reference for myself, but hopefully it proves useful for others as well.

[Remote Access](#remote-access)  
[Remote Execution](#remote-execution)  
[Scheduled Tasks Creation](#scheduled-task-creation)  
[Scheduled Tasks Listing](#scheduled-tasks-listing)  
[Service Creation](#service-creation)  
[Service Investigation](#service-investigation)  
[WMIC Service Listing](#wmic-service-listing)  
[WMIC Remote Process Creation](#wmic-remote-process-creation)  
[WMIC Process Listing](#wmic-process-listing)  
[WMIC System Arch and Resource Queries](#wmic-system-arch-and-resource-queries)   
[WMIEvtConsumers](#wmievtconsumers)  
[Convert SID to Username With PowerShell](#convert-sid-to-username-with-powershell)   
[PowerShell Remoting](#powershell-remoting)  
[Kansa](#kansa)  
[Delete Directory With CB Response Go-Live](#delete-directory-with-cb-response-go-live)   
[Hunting Persistent Malware](#hunting-persistent-malware)  
[Autoruns](#autoruns)  
[Minifilter Driver Management Operations and Troubleshooting Using fltmc](#minifilter-driver-management-operations-and-troubleshooting-using-fltmc)   
[Netstat Hacks](#netstat-hacks)  
[Sneaky Evil Hacks](#sneaky-evil-hacks)  
[Configure Windows host as WPA2-PSK Access Point](#configure-windows-host-as-wpa2-psk-access-point)  
[General SysAdmin Stuffs](#general-sysadmin-stuffs)   
[Generate CSR](#generate-csr)
# Remote Access
### Map admin share using net.exe
```
net use z: \\host\c$ /user:domain\username <password>
```

# Remote Execution

### Scheduled Tasks Creation
```
# Deprecated
at.exe \\host 12:00:00 "c:\temp\maltask.exe"
# Uses type 3 login
schtasks.exe /CREATE /TN taskname /TR c:\temp\malware.exe /SC once /RU "SYSTEM" /ST 12:00:00 /S host /U username
```

### Scheduled Tasks Listing
```
# Uses type 3 login
schtasks.exe /S HOSTNAME /FO CSV /v | ConvertFrom-Csv
```

### Service Creation
```
sc \\host create servicename binpath="c:\temp\malware.exe"
sc \\host start servicename
```

### Service Investigation
```
tasklist /svc
sc config <svc name> type=own
net stop <svc name> && net start <svc name>
```
List services in each svchost block, break a service out of its container, and restart that service in its own container. Great for troubleshooting resource hogging services, discovering services to exploit, or hunting for malware.

If service keeps crashing then check that services recovery options - could be running another program instead of restarting (Kansa module Get-SvcFail.ps1 useful here)

### WMIC Service Listing
```
# Only started services
wmic /node:"HOSTNAME" service where started=true get name,startname,description /format:csv > started_services.csv
# All installed serviecs
wmic /node:"HOSTNAME" service get name,startname,description /format:csv > services.csv
```

### WMIC Remote Process Creation
```
wmic /node:host process call create "malware.exe"
Invoke-WmiMethod -Computer host -Class Win32_Process -Name create -Argument "c:\temp\malware.exe"
```

### WMIC Process Listing
```
wmic process list full
```

### WMIC System Arch and Resource Queries
```
# Query total system memory
wmic computersystem get TotalPhysicalMemory
# Query OS architecture
wmic OS get OSArchitecture
```   

### WMIEvtConsumers
- Event Filter = Trigger condition
- Event Consumer = Script or executable to run
- Binding = Filter + Consumer
- Usually saved to .MOF file
- Triggers can be basically ANYTHING

### Convert SID to Username With PowerShell
```
([System.Security.Principal.SecurityIdentifier]($SID)).Translate([System.Security.Principal.NTAccount]).Value 
```

### PowerShell Remoting
Create new PS Session on target, transfer files to or from target, and kill session.
```
$TargetSession = New-PSSession -ComputerName <hostname>
Copy-Item -ToSession $TargetSession -Path <src> -Destination <dst> -Recurse
Copy-Item -FromSession $TargetSession -Path <src> -Destination <dst> -Recurse
Remove-PSSession <id>
```
Enter new session and execute malware on target.
```
Enter-PSSession -ComputerName host
Invoke-Command -ComputerName host -ScriptBlock {Start-Process c:\temp\malware.exe}
```

### Kansa
(placeholder)

### Delete Directory With CB Response Go-Live
```
execfg cmd /c rd /s /q $DIRECTORY
```

# Hunting Malware

### Autoruns
Filter output based on publisher, eliminate verified known publishers, and examine what remains.
```
C:\> autorunsc.exe -accepteula [options] > autoruns.csv

-accepteula    Automatically accept Microsoft software license
-a *           Show all entries
   b           Boot execute
   c           Codecs
   d           Appinit DLLs
   e           Explorer add-ons
   g           Sidebar gadgets (Vista+)
   h           Image hijacks
   i           Internet Explorer add-ons
   k           Known DLLs
   l           Logon startups (default for autorunsc)
   m           WMI entries
   n           Winsock protocol and network providers
   o           Office add-ins
   p           Printer monitor DLLs
   r           LSA security providers
   s           Autostart services and non-disabled drivers
   t           Scheduled tasks
   w           Winlogon entries

-h             Show file hashes
-m             Hide Microsoft signed entries
-s             Verify digital signatures

-t             Show timestamps in normalized UTC
-c             Print output as CSV
-ct            Print output as tab-delimited values
-x             Print output as XML

-vt            Accept VirusTotal terms of service
-v[rs]         Query VirusTotal based on file hash
   r           Opens reports for files with non-zero detection
   s           Uploads files that have never been seen by VirusTotal

-z             Specifies offline Windows system to scan

-nobanner      Do not display startup messages

# Live System Example:
C:\> autorunsc.exe -accepteula -a * -h -s -c -m -vr > autoruns.csv

# Offline System Example:
C:\> autorunsc.exe -accepteula -a * -h -s -c -m -vr -z 'E:\Windows' 'E:\Users' > autoruns.csv

# NOTE: The first time you run autorunsc.exe with VirusTotal flags it will open a browser displaying their terms and conditions. The terminal will display "Enter 'y' to agree:", so don't redirect to a csv file the first time you run it or you won't be able to enter y.
# ANOTHER NOTE: when analyzing offline systems there are a few idiosyncrasies to be aware of
    # Always run from an Administrator shell
    # Always mount your image with WRITE options or the program won't be able to load certain hives
    # For some reason even if the image is mounted with WRITE options, and autorunsc.exe is run as Administrator, it will SOMETIMES report that it can't load the SOFTWARE hive. If I open the image in Autoruns.exe (the GUI version) and then try autorunsc.exe again, it runs properly... *shrugs*
```
Additionally, the GUI version of Autoruns is also available, but I prefer the CLI version because it enables you to programmatically manipulate output. The GUI output includes color coordination - The colors mean the following.
- Pink - No publisher information was found, or if code verification is on, the digital signature either doesn't exist, or doesn't match, or there is no publisher information.
- Green - Used when comparing against a previous set of Autoruns data highlight items that weren't already there.
- Yellow - The startup entry is there, but the file or job it points to doesn't exist anymore.   

[SANS Autoruns writeup](https://sans.org/reading-room/whitepapers/malicious/utilizing-autoruns-catch-malware-33383)   
[HowToGeek Autoruns GUI reference / writeup](https://www.howtogeek.com/school/sysinternals-pro/lesson6/)

### Minifilter Driver Management Operations and Troubleshooting Using fltmc
Used to load and unload minifilter drivers, attach minifilter drivers to volumes or detach them from volumes, and enumerate minifilter drivers, instances, and volumes. I once had an issue where Sysmon and a particular EDR product weren't playing well together; this tool assisted the investigation.   
```
C:\> fltmc help

Valid commands:
    load        Loads a Filter driver
    unload      Unloads a Filter driver
    filters     Lists the Filters currently registered in the system
    instances   Lists the Instances for a Filter or Volume currently registered in the system
    volumes     Lists all volumes/RDRs in the system
    attach      Creates a Filter Instance to a Volume
    detach      Removes a Filter Instance from a Volume
```
[Great fltmc.exe reference / writeup](https://blogs.msdn.microsoft.com/ntdebugging/2013/03/25/understanding-file-system-minifilter-and-legacy-filter-load-order)

### Netstat Hacks
```
netstat -nabo 1 | find "<IPADDR or PORT>"
```

# Sneaky Evil Hacks

### Configure Windows host as WPA2-PSK Access Point
```
netsh wlan set hostednetwork mode=allow ssid=<MYSSID> key=<MYPASSWORD> && netsh wlan start hostednetwork
```

# General SysAdmin Stuffs

### Generate CSR
Add the following to a config file (san.cnf)
```
[ req ]
default_bits       = 2048
distinguished_name = req_distinguished_name
req_extensions     = req_ext
[ req_distinguished_name ]
countryName                 = Country Name (2 letter code)
stateOrProvinceName         = State or Province Name (full name)
localityName               = Locality Name (eg, city)
organizationName           = Organization Name (eg, company)
commonName                 = Common Name (e.g. server FQDN or YOUR name)
[ req_ext ]
subjectAltName = @alt_names
[alt_names]
DNS.1   = hostname
DNS.2   = hostname.domain.com
```
Run the following command
```
# Replace the following variables respectively: $DOMAIN, $COUNTRYABRV, $STATEABRV, $ORG, $HOSTNAME
openssl req -out $DOMAIN.csr -newkey rsa:2048 -nodes -keyout $DOMAIN.key -config san.cnf -subj '/C=$COUNTRYABRV/ST=$STATEABRV/O=$ORG/CN=$HOSTNAME.$DOMAIN.com'
```
