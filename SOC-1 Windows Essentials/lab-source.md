User & Account Management
Step 1: Understand User Accounts on Windows

Windows supports different account types:

Local User Accounts : Exist only on the machine
Administrator Accounts : Have full system control
Service Accounts : Used by services/applications
Step 2 : Identifying local users and groups

Press Win + R > type lusrmgr.msc > press Enter
Content Image
Navigate to Local Users and Groups > Users
Content Image

Here you can see all the local users.

Navigate to Local Users and Groups > Groups
Content Image
Here you can see all the local groups.

This section highlights some of the key Windows user groups :

The Administrators group is the most powerful built-in group in Windows. Members of this group have full control over the system, including the ability to install and remove software, modify system settings, manage users and groups, and access all files on the machine.

The Remote Desktop Users group allows users to log in to a Windows system remotely using Remote Desktop Protocol (RDP). Members of this group can connect to the machine over the network and interact with the desktop environment.

The Remote Management Users group is intended for users who need to manage or monitor a system remotely without full administrative access. Members can connect using Windows Remote Management (WinRM), PowerShell Remoting, and certain management tools to retrieve system information or perform limited administrative tasks.

The Users group is the default group for standard accounts in Windows. Members can run installed applications, use network resources, and manage their own files and settings. This group is designed for everyday use and represents the principle of least privilege.

The Backup Operators group is designed for accounts that perform backup and restore operations. Members can back up and restore files regardless of file permissions, meaning they can access data that would normally be restricted.

Step 3 : Check if an account is disabled

Right-click a user > Properties
Under General tab, see if “Account is disabled” is checked.
Content Image

We can clearly see that the user student is not disabled.

Step 4: Password policies define how strong passwords must be.

Press Win + R > Type secpol.msc > Press Enter
Content Image
Now navigate: Local Security Policy > Account Policies > Password Policy
Content Image

Here we can see the password policies that are currently set.

We can see current password policies that are configured on the system are extremely weak and does not enforce basic security standards. The minimum password length is set to 0 characters, meaning users can create accounts with blank or empty passwords. This completely removes password-based authentication protection and allows unauthorized access with no effort. Additionally, password complexity requirements are disabled, so even if a password is set, it can be simple, predictable, and easily guessable.

The policy also specifies a maximum password age of 0 and a minimum password age of 0 days, which means passwords never expire and users are not required to change them periodically. As a result, compromised passwords can remain valid indefinitely, increasing the risk of long-term unauthorized access.

We should enforce a strong password policy to ensure passwords are complex, unique, and resistant to guessing and credential-based attacks.

Step 5: Account Lockout Policy protect against brute-force attacks.

Now navigate: Local Security Policy > Account Policies > Account Lockout Policy
Content Image

Content Image

The current account lockout policy on the system is effectively disabled, making it highly vulnerable to password-guessing and brute-force attacks. The account lockout threshold is set to 0 invalid logon attempts, which in Windows means that accounts will never be locked out regardless of how many failed login attempts occur. As a result, attackers can repeatedly attempt to guess passwords without triggering any defensive response.
We should enforce an account lockout policy to protect ourselves from brute-force and repeated login attempts by automatically locking accounts after multiple failures.

Process & Service Investigation
Step 1 : Open Task Manager (Ctrl + Shift + Esc).

Task Manager provides real-time visibility into running processes and resource usage.

Content Image

Step 2 : Right click on details.

Content Image

Now we can see process ID (PID), CPU, memory, user name etc for each process.

Step 3 : Right click on a process and click on properties.

Content Image

Here you can see the file location and other properties for that process inluding its digital signature

Step 4 : Open Services

Win + R > services.msc
Content Image
Content Image

This will display all the services running on the system along with their status and startup type.

The Startup Type defines how and when a service starts. Understanding these startup types is important for system administration, performance tuning, and security :

A service set to Automatic starts automatically during system boot without any user intervention. These services are considered essential for the normal operation of Windows or critical applications, such as networking, security, or core system functions.

Automatic (Delayed Start) services also start automatically, but only after the system has finished booting and critical services are already running.

A Disabled service cannot be started automatically or manually until it is explicitly re-enabled by an administrator. Windows will completely prevent the service from running, regardless of system or application requests.

A service with the Manual startup type does not start automatically when Windows boots. Instead, it starts only when required by the system, an application, or when manually started by an administrator.

Step 5 : Open Startup Folder

Win + R > shell:startup
Content Image

Content Image

Adversaries commonly achieve persistence on Windows systems by abusing startup folders which are legitimate mechanisms designed to automatically launch programs when a user logs in or when the system boots. By adding a malicious executable or script to these locations, attackers ensure that their payload is executed every time the system starts or a user signs in, allowing long-term access even after reboots.

There are separate startup folders for individual users and for all users on the system. The user-specific startup folder executes only when that user logs in, while the system-wide startup folder executes regardless of which user signs in.

Step 6: Open Task Scheduler

Win + R → taskschd.msc
This will open Windows Task Scheduler. It displays all scheduled tasks configured on the system. Tasks can be triggered by logon, startup, time-based events, or system actions.

Content Image

Content Image

Step 8: Let's now inspect an already running task and inspect it.

On the left panel, expand Task Scheduler Library > Microsoft > Windows
Here we can browse folders to find task
Content Image

Content Image

Content Image

Here we can clearly see the name of the task along with its status, what actually triggers it and other details.

Step 8: Let's now see how we can schedule a task.

Content Image

Click on the create task button on the right panel.

Content Image

We can see Create task modal opened up

Content Image

In General tab, we give name to our task. Here we have other options too, includidng when to run it, only if user logged in or in both logged in/off.

We can also provide a description for our task.

Content Image

In the trigger tab, we set the schedule and frequency for the task

Content Image

In the actions tab, we provide the actual program and the script for the task. Here we can also provide arguements

We also have conditions tab, where we check like if we want the task to run only if the computer is idle.

Windows Logging & Event Viewer
Windows does not log everything by default. Proper auditing is required capture security-relevant events required for monitoring, including logon activity, account changes, and process creation.

So let's enable window event logging first

Step 1: Open Local Security Policy

Press Win + R > secpol.msc > Press Enter
Content Image
Step 2: Navigate to Audit Policy

Local Policies > Audit Policy
Content Image

The Audit Policy found under Local Policies is the legacy auditing mechanism in Windows. It provides broad audit categories such as logon events, object access, and policy changes, but does not offer fine-grained control. When enabled, it records high-level success or failure events without specifying the exact type of activity that occurred. Because of this, the logs generated can be insufficient for detailed investigations.

Step 3: Open Advanced Audit Policy

Navigate to :

Security Settings > Advanced Audit Policy Configuration > Audit Policies
Content Image

The Advanced Audit Policy Configuration is a newer and more detailed auditing framework introduced to provide more control over security logging. Instead of broad categories, it breaks auditing into specific subcategories such as credential validation, logon type, privilege use, process creation, and file access. This allows administrators to precisely control what is logged.

When Advanced Audit Policy is configured, it overrides the legacy Audit Policy settings if the system is configured to enforce advanced auditing.

Step 4: Enable High-Value Subcategories

Enable Success and Failure for: - Logon / Authentication - Credential Validation - User Account Management - Security Group Management

Content Image

Content Image

Content Image

Content Image

Content Image

Let's now open Event viewer and observe the logs

Step 1: Launch Event Viewer

Press Win + R > eventvwr.msc > Press Enter
Content Image

Content Image

This will open up Windows Event Viewer.

Step 2: Understand the Log Categories

Event Viewer > Windows Logs
Content Image
Here we'll see different log types : Application logs, Security logs, System logs etc.

Step 3: Open Security Logs

Click Windows Logs → Security
Here we'll see the events related to security. Some of them are mentioned below:
Successful logon       -     4624

Failed logon           -     4625

User account created   -     4720

User added to group    -     4732

LogOff                 -     4634
Content Image

Here we can see the Event ID is 4720 which indicates that a new user account was created on the system. We can also see the that the new account is created by Administrator and name of the new account is John123.

Content Image

Event ID 4624 indicates that an account was successfully logged on to the system. We can also see that the account used for login was System.

Content Image

Event ID 4732 indicates that a user account was added to a security-enabled local group. The event records which account performed the action (Administrator), which user was added (John123), and the target group (Guests).

Content Image

Event ID 4634 indicates that a user account( in this case Administrator) was successfully logged off from the system. The event also records session ID helping track session termination and user activity timelines.

Step 3: Open Application logs

Windows Logs → Application
The Application log contains events logged by applications and services. It includes information messages, warnings, and error events such as application crashes. These logs help identify issues related to software configuration and runtime failures.
Step 4: Open System logs

Windows Logs → System
Here we'll find the logs related to system startup and shutdowns, service installation etc.
Content Image

Event 7045 indicates a new service either created or installed on the system.

Content Image

Event ID 6006 means the Windows Event Log service was stopped. This happens during a clean system shutdown or restart.

Step 5 : Open Sysmon logs

Applications and Services Logs → Microsoft → Windows → Sysmon → Operational
Sysmon (System Monitor) is a Windows system service and driver from Microsoft Sysinternals that provides high-detail security logging far beyond default Windows logs in a very structured manner.

Content Image

When a new Process is created, Sysmon generates an event ID 1. This can be used for detecting malware execution and suspicious processes.

Content Image

Event ID 6 is logged when a kernel-mode driver is loaded. It is used for detecting malicious or unsigned drivers.

Content Image

Event ID 22 is logged when a process performs a DNS query. It can be used for detecting malicious domains and suspicious network activity.

Step 6 : Filtering and searching logs

Right-click a log >  Select Filter Current Log
Here we'll have filter options like Event IDs, time range.

Content Image

Now let's try to filter the event by Event ID. Here we are using Event ID 4720 to filter the events which is for user Account creation

Content Image

We can see that we have two events with Event ID 4720 that have been filtered Out.

We can also filter the events by User, Event level, time range and Task category.

Windows Command Line Essentials (PowerShell)
Step 1 : List local users

Get-LocalUser
Content Image
This command returns all local user accounts present on the system, helping in identifying unauthorized or unexpected accounts.

Step 2 : List local groups

Get-LocalGroup
Content Image
This command returns all local security groups on the system, which are used to assign privileges and permissions.

Step 3 : Check group membership

Get-LocalGroupMember <group_name>
Content Image
This command returns all users and accounts that belong to the specified local group, helping detect privilege abuse.

Step 4 : List running processes

Get-Process
Content Image
This command returns all currently running processes along with their process IDs and resource usage.

Step 5 : Find a specific process by name

Get-Process -Name "<process-name>"
Content Image
Step 6 : List running services

Get-Service
Content Image
This command returns all Windows services and their current state (running or stopped), useful for identifying suspicious or unauthorized services.

Step 7 : Find a specific service by name

Get-Service -Name "<service_name>"
Content Image
Step 8 : Finding more details about a service

Get-CimInstance Win32_Service | Where-Object {$_.Name -eq "<service_name>"} | Select-Object Name, DisplayName, State, StartMode, StartName, PathName
Content Image

Step 9 : List all Schedule tasks

Get-ScheduledTask
Content Image

Step 10 : Filter tasks by name pattern:

Get-ScheduledTask | Where-Object {$_.TaskName -like "*update*"}
Content Image

Step 11 : List available logs

Get-WinEvent -ListLog *
Content Image
This command returns all Windows Event Logs available on the system, which are used for security monitoring and investigation.

Step 12 : Filter Event by EventID + LogName

Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4720} -MaxEvents 20 |
Select-Object TimeCreated, Id, ProviderName, Message
Content Image

Step 13 : Filter Event by EventID + Target Account

Get-WinEvent -LogName Security -FilterXPath "*[System[(EventID=4624)]] and *[EventData[Data[@Name='TargetUserName']='administrator']]" |
Select-Object TimeCreated, Id, Message
Content Image

Windows Registry Basics
The Windows Registry is a hierarchical database that stores:

System configuration

User settings

Application behavior

Security and startup configurations

Step 1 : Accessing the registry

Opening Registry Editor

Press Win + R > type regedit > Enter
Content Image

Registry Structure

HKEY_LOCAL_MACHINE (HKLM) - System‑wide settings

HKEY_CURRENT_USER (HKCU) - Current user settings

HKEY_USERS (HKU) - All loaded users

HKEY_CLASSES_ROOT (HKCR) - File associations

HKEY_CURRENT_CONFIG (HKCC) - Current hardware profile

Content Image

Registry Data Types

REG_SZ - String value - File paths, commands

REG_DWORD - 32‑bit number - Enable/Disable flags

REG_BINARY - Binary data - Encoded config

REG_MULTI_SZ - Multiple strings - Lists of paths

Registry Inspection Using PowerShell

The Run registry (often called Run keys) is a set of keys inside window registry used to automatically start programs when Windows boots or when a user logs in. It is a legitimate Windows feature designed to launch required applications such as system utilities, security software, or user applications without manual intervention. When a program is added to a Run registry key, Windows executes that program automatically at logon. From an attacker’s perspective, adding a malicious executable, script, or command to a RUN key can ensure their payload is launched every time the system starts or a user signs in.

Commonly abused RUN registry keys include HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run and HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run. Entries under the HKCU hive execute in the context of the logged-on user and do not require administrative privileges, making them particularly attractive for attackers who have gained limited user access. In contrast, entries under HKLM affect all users on the system but typically require elevated privileges to modify, allowing attackers to establish broader and more persistent control.

Attackers may also use related keys such as RunOnce, which executes a program only once at the next logon, often to complete post-exploitation tasks

Step 1 : Check autoruns (system) This key contains programs that run for any user when Windows starts.

Get-ItemProperty "HKLM:\Software\Microsoft\Windows\CurrentVersion\Run"
Content Image
Step 2 : Check a specific service This key stores configuration details for Windows services and drivers. Each subkey represents one service.

Get-Service
Get-Item "HKLM:\SYSTEM\CurrentControlSet\Services\<ServiceName>"
Content Image

Content Image

Conclusion
This Windows Essentials lab provided hands-on exposure to core Windows security and administration concepts, including user and group management, password and account lockout policies, auditing, service and startup behavior, scheduled tasks, and event log analysis. By exploring both legitimate administrative functionality and how these mechanisms can be abused by attackers, the lab reinforced the importance of secure configuration and continuous monitoring.

Through practical exercises, the lab demonstrated how weak policies and misconfigurations can introduce security risks, while proper controls such as strong password enforcement, account lockout policies, auditing, and task monitoring help mitigate common attack techniques like brute-force attacks and persistence mechanisms. Overall, this lab strengthened the understanding of Windows internals from both an administrative and security perspective, highlighting the critical role of defense-in-depth in securing Windows systems.