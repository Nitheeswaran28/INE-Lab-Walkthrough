Solution
Step 1: Open the lab link to access the Attacker (Kali), Ubuntu (Kibana), and Windows Server 2019 machines.

Attacker Machine

Content Image

SOC Machine

Content Image

Target Machine

Content Image

Task 1: Start HELK system for Kibana access.
First, we need to start the helk system for advanced log analytics and threat hunting.

Step 2: The .yaml file is located in /opt/HELK/docker folder. We will start the HELK setup by running the helk_install.sh bash script.

Commands:

cd /opt/HELK/docker/
./helk_install.sh
Content Image

Content Image

Note: Wait for 5 minutes to get all the services up

Running the Firefox Browser and accessing the Kibana.

URL: https://localhost

[Certificate-warning instruction omitted.]

Username: helk

Password: [redacted]

Content Image

Step 3: We can notice that the Kibana dashboard is up and running perfectly. Now, let's move ahead and configure the windows machine for advanced logs.

Task 2: Enable Detailed Tracking
Audit Process Creation
Include the Command line in process creation events
Audit Process Termination
Audit - Force audit policy subcategory settings
Enabling Command Line Logging with Arguments
Enable Powershell Modules and Scripting blog Logging
Create a preference variable for Windows PowerShell Event Log
Enabling Task Scheduler History/Logging
etc.
Switch to the Windows (Target) machine and start gpedit.msc and Go to: Windows Settings -> Security Settings -> Advanced Audit Policy Configuration -> System Audit Policies - Local Group Policy Object -> Detailed Tracking

Content Image

Enable Audit Process Creation
Let's enable Audit Process Creation & Audit Process Termination

Content Image

Enable Audit Process Termination
Content Image

Content Image

Audit - Force audit policy subcategory settings
Now, we will enable Force audit policy subcategory settings

Go to: Windows Settings -> Security Settings -> Local Policy -> Security Options -> Audit - Force audit policy subcategory settings

Content Image

Content Image

About: Audit - Force audit policy subcategory settings

Audit: Force audit policy subcategory settings (Windows Vista or later) to override audit policy category settings. Windows Vista and later versions of Windows allow audit policy to be managed in a more precise way using audit policy subcategories. Setting the audit policy at the category level will override the new subcategory audit policy feature. Group Policy only allows audit policy to be set at the category level, and existing group policy may override the subcategory settings of new machines as they are joined to the domain or upgraded to Windows Vista or later versions. To allow audit policy to be managed using subcategories without requiring a change to Group Policy, there is a new registry value in Windows Vista and later versions, SCENoApplyLegacyAuditPolicy, which prevents the application of category-level audit policy from Group Policy and from the Local Security Policy administrative tool. If the category-level audit policy set here is not consistent with the events that are currently being generated, the cause might be that this registry key is set.

Enabling Command Line Logging with Arguments
Now, we will enable Command Line Logging with Arguments in the group policy.

Go to: Windows Settings -> Administrative Templates -> System -> Audit Process Creation

Content Image

Content Image

Content Image

Enable Powershell Modules and Scripting blog Logging
We will enable multiple scripts in Windows Powershell components.

Turn on Module Logging
Turn on PowerShell Script Block Logging
Go to: Windows Settings -> Administrative Templates -> Windows Components -> Windows Powershell

Content Image

Turn on Module Logging
Content Image

Turn on PowerShell Script Block Logging
Content Image

Create a preference variable for Windows PowerShell Event Log
Selecting Events for the Windows PowerShell Event Log

You can use the event log preference variables to determine which events are recorded in the Windows PowerShell event log.

There are six event log preference variables, two variables for each of the three logging components: the engine (the Windows PowerShell program), the providers, and the commands. The LifeCycleEvent variables log normal starting and stopping events. The Health variables log error events.

In this case, we will create a profile.ps1 file in the powershell directory.

Access: C:\Windows\System32\WindowsPowerShell\v1.0 folder and create and add below two log variables in the profile.ps1 file.

$LogCommandLifeCycleEvent = $true
$LogCommandHealthEvent = $true
Content Image

Now, confirm whether these variables are enabled by running the powershell terminal.

Content Image

Enabling Task Scheduler History/Logging
Now, we will enable the Task Scheduler History.

We can run the below powershell commands to enable the task scheduler history.

$logName = 'Microsoft-Windows-TaskScheduler/Operational'
$log = New-Object System.Diagnostics.Eventing.Reader.EventLogConfiguration $logName
$log.IsEnabled=$true
$log.SaveChanges()
Content Image

Now, open the task scheduler and verify that the task scheduler history option is enabled.

Content Image

We have configured the windows target machine based on our objectives and advanced logging. One can make more changes based on the need and output.

After all the changes, please remember to restart the Target Machine.

Command:

Restart-Computer -Force
Wait for 2-3 minutes and click on Reconnect

Task 3: Configure Sysmon on Windows Server
Step 4: What is sysmon?

Sysmon is a Windows system monitoring tool that allows administrators to monitor and log system activity to the Windows event log. It can track a variety of system events, including process creation and termination, network connections, and changes to file creation time. Sysmon also includes a driver component that monitors the operating system at a low level, providing additional information not available through event logging.

Sysmon is developed by Microsoft Sysinternals, and can be used to detect and investigate malicious activity on a Windows system, such as reconnaissance and data exfiltration. Because it logs detailed information about system activity, Sysmon can be used to track down the source of security incidents, such as intrusions and malware infections. Additionally, Sysmon can be used to monitor the behavior of legitimate software and to detect any abnormal activity.

The tool can be configured using a XML configuration file, This allows an administrator to customize what events are logged, as well as specify filters to exclude certain types of activity from being logged. Sysmon supports both 32-bit and 64-bit versions of Windows and it is a command line tool.

Sysmon is a powerful tool for security professionals, but it does produce a high volume of log data, so it is important to have a good logging and event management infrastructure in place when using it. It's also important to note that running sysmon for long time may affect the performance of the system.

Source: https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon

The sysmon executable and its .xml file are provided to you in the C:\Users\Administrator\Documents folder. The .xml file is from SwiftOnSecurity - sysmon-config | A Sysmon configuration file for everybody to fork. This sysmonconfig-export.xml file is the configuration file template with default high-quality event tracing.

Let's start the powershell terminal and run the below command to enable sysmon logs.

Commands:*

C:\Users\Administrator\Documents\Sysmon.exe -accepteula -i C:\Users\Administrator\Documents\sysmonconfig-export.xml
Get-Service -Name Sysmon
Content Image

We have installed sysmon as a service and used the sysmonconfig-export.xml file for high-quality log event tracking.

Learn More About sysmon event logs: https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon

Task 4: Configure winlogbeat on Windows Server Machine
Step 5: All the important configurations are done on the windows machine. Now, we want to send all the logs to the Elasticsearch server using the winlogbeat utility.

The winlogbeat is located in the C:\Users\Administrator\Desktop\Tools\winlogbeat folder.

First, we need to edit the winlogbeat.yml file to add the HELK (SOC Machine) IP address.

Switch to the Ubuntu SOC Machine and check the IP address.

Command:

ifconfig ens5
Content Image

The Ubuntu SOC machine IP address is: 10.0.21.29

Again Switch to the Windows Machine and modify the winlogbeat.yml file.

From:

hosts: ["<HELK-IP>:9092"]
To:

hosts: ["10.0.21.29:9092"]
Content Image
Save the configuration and run the install-service-winlogbeat.ps1 powershell script to install the winlogbeat as a service.

Commands:

powershell -ep bypass
cd C:\Users\Administrator\Desktop\Tools\winlogbeat
.\install-service-winlogbeat.ps1
Start-Service -Name winlogbeat
Get-Service -Name winlogbeat
Content Image

The winlogbeat service is up and running.

Note Once you start the winlogbeat service, the RDP session might be disconnected; in that case click on reconnect.

We are all set to monitor the windows machine. Now, let's perform all the attacks from the Kali machine.

Tasks 5-7: Review the recorded HFS compromise, persistence, and credential-access scenarios.
[Offensive execution procedures omitted.]

Task 8: Detect all the attack's first foothold, maintaining access and detecting mimikatz credentials dump.
Detecting Attacks on Kibana
Step 8: Now, Let's access the Kibana dashboard, and all the logs should be present there.

URL: https://localhost

[Certificate-warning instruction omitted.]

Username: helk

Password: [redacted]

Content Image

First, let's access the Kibana Dashboard.

Content Image

We can notice multiple dashboards has been created to analyze the sysmon logs quickly.

Click on the Sysmon Dashboard (HELK).

We can notice 116 counts are present there.

Note: In your case, the counts would be different. Also, we can observe that wmiadap.exe ran 33 times on the target machine.

Content Image

Also, if we scroll down, we can notice all the sha256 hashes of the executables. These hashes help check the executable authenticity.

Content Image

Now, click on the Sysmon Network Dashboard (HELK).

We notice that 38 counts are present. Also, we found all the destination IP addresses.

Content Image

We can observe that some unknown executables are also running on the target machine. This is the same executable that is used for maintaining access. Also, using the wscript.exe executable is only sometimes normal. It's always a good idea to investigate these processes further.

Content Image

Now, let's go to the Kibana Discover and find how the hfs.exe application is exploited.

Content Image

Search for "hfs.exe", and we can see all results to this executable only.

Content Image

If we look closely at the log, we can observe that wscript.exe ran with hfs.exe as Process Command.

C:\Windows\System32\wscript.exe" //B //NOLOGO %TEMP%\tlqjxleo.vbs
Let's change the search with "hfs.exe" and "wscript.exe"

Content Image

This confirms that a .vba script is running through the hfs.exe executable.

Also, while running the persistence script, we noticed that a .vbs script was placed on the target machine. Finding these scripts is also accessible by filtering the file type to .vbs

.vbs
Content Image
We found the malicious *.vbs** script location.

c:\users\admini~1\appdata\local\temp\2\douwxodqwof.vbs
The .vba script dumps an executable file in the temp directory.
C:\Users\ADMINI~1\AppData\Local\Temp\2\radCB854.tmp\GtrkSyafTdtynCg.exe
Now, detecting the usage of mimikatz.exe is also very easy. We know the executable name. Let's search with the name (mimikatz), and we will get all the events associated with it.
Content Image
Based on this output one can investigate further, we haven't perforced any advance attacks, we have just extracted logon credentials. One can check the security event IDs to confirm that it has accessed to the sensitive processes i.e lsass.exe and other. To detect its usage in Kibana, we can specifically looking for events related to Mimikatz. The specific event IDs and log sources will depend on the version of Windows and Mimikatz being used. You can also search for specific Mimikatz commands or other indicators of its usage in process execution logs, network traffic logs, or other relevant data sources. It is also possible to use endpoint security tools that can detect and alert on the use of Mimikatz or other malicious tools on a system. One can modify to the HELK system according to the need and requirements. This allows a Blue teamer to quickly check active threats on the windows machines. One can explore more, by performing more attacks on the target windows machine.