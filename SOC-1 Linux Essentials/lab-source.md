User & Account Management
Step 1: Understanding User Accounts on Linux

Linux supports:

Root user – Superuser with unrestricted access to the system, able to modify any file, change configurations, and manage all users and services.

Local users – Regular user account created by administrator for logging in and performing day-to-day tasks on the system.

System / Service accounts – A non-login account created for running system services and background processes

Step 2: Identifying users

cat /etc/passwd
Content Image
Each entry is in the format : username:x:UID:GID:comment:home_directory:login_shell

UID -> user_id | GID -> group_id

Now in Linux, a user’s login shell determines whether they can get an interactive session. If a user’s shell is: - /bin/bash - /bin/sh - /bin/zsh

They can log in and execute commands

If the shell is:

/usr/sbin/nologin
/bin/false
They cannot get an interactive login

Step 3: Understanding User Groups on Linux

Primary group: The main group assigned to a user. Files created by the user inherit this group.

sudo group: Members can run commands with root privileges using sudo.

adm group: Allows access to system logs and monitoring files.

root group: The group associated with the root user, with full system control.

sys group: A legacy system group historically used for owning core system and kernel-related files. On modern Linux systems, it usually has no active role and exists mainly for backward compatibility.

Step 4: Identifying groups

cat /etc/group
Content Image

Each entry is in the format : group_name : password : GID : members

Step 5: Check if a user is locked or disabled

grep <user_name> /etc/shadow
Content Image

Locked accounts start with !

We can clearly see that the account news is locked.

Step 6: Understanding Linux Password & Authentication Controls

In Linux, password policy and authentication behavior are primarily controlled by two configuration files. The /etc/login.defs file defines system-wide password policies such as password aging, expiration, and minimum length, while the PAM configuration files (primarily /etc/pam.d/common-password and related files) control how users are authenticated and enforce password complexity rules. Together, these files determine how passwords are created, validated, and managed across the system.

cat /etc/login.defs
Now let's go through some important sections of this file
Content Image

What it does : - Enables logging of failed login attempts - Data stored in /var/log/faillog

It is useful for:

Brute-force detection
Credential abuse investigations
It is also important to notice that this option conflict with the pam_tally PAM (Pluggable Authentication modules) module, since both of them do the same job which can lead to inconsistent and duplicate tracking.

FAILLOG_ENAB → legacy, non-PAM based tracking
pam_tally / pam_tally2 / pam_faillock → PAM-based, modern approach
Content Image

Here LOG_UNKFAIL_ENAB is disabled which is for logging Failed login attempts for nonexistent users.

Now disabling this can be risky because it can Reduces visibility into : - Username enumeration attacks - Password spraying attempts

Content Image

LOG_OK_LOGINS is for logging successful login attempts. Right bnow we can see, it is disabled which can reduce audit visibility.

Content Image

These two options SYSLOG_SU_ENAB and SYSLOG_SG_ENAB are useful for monitoring Privilege escalation.

It is useful for : - Tracks privilege escalation attempts - Useful for detecting lateral movement

Content Image

These three options are very critical because if we llok at what they are set to as of now, it means : - Passwords effectively never expire - Users can change passwords immediately - Warning only 7 days before expiration (irrelevant if never expires)

Now in case if credentials got leaked, these values will lead to an increasing risk of credential reuse and prolonged unauthorized access.

Content Image

With these options, we can set the number of times login attempts can be made and login session time outs.

cat /etc/pam.d/common-password
Now let's go through the important section for this file
Content Image

This password [success=1 default=ignore] pam_unix.so obscure sha512 single line is the main password policy.

pam_unix.so is the core authentication module. It - Handles local user passwords - Reads/writes password hashes in /etc/shadow

sha12 here means that the passwords are hashed using SHA-512

obscure prevents : - Password == username - Simple patterns - Very weak passwords

This [success=1 default=ignore] is the PAM control flag which means - If pam_unix.so succeeds, PAM skips the next module - If it fails, continue normally

We also have password requisite pam_deny.so in here which is a fallback denial module. This module is a safety net which prevent password changes if something breaks upstream.

Now what is missing from this file is pam_pwquality.so module. This is a PAM module used to enforce password complexity and strength rules.

What this module does : - Checks password length - Enforces complexity requirements (uppercase, lowercase, digits, special characters) - Prevents weak or easily guessable passwords

This module is configured in /etc/pam.d/common-password file.

Investigating Processes and Services
Step 1: View all the running processes

ps aux
This command lists all running processes on the system.
Content Image

Here for each process we'll get the process id, account running that process, CPU & memory usage, what program is actually running, start time for the processes.

Step 2 : Investigate a specific process

Pick a process ID

ps -p $PID -o pid,ppid,user,cmd
For this, let's open featherpad. This will start a new process for featherpad, then we can copy that iD.
Note : In your case process_id can be different.

Content Image

Check executable path:

readlink -f /proc/$PID/exe
Content Image

Step 3 : Filter a specific process

ps aux | grep -i <process_name>
Content Image

Step 4 : List running services

systemctl list-units --type=service --state=running
Content Image

Here we can see the list of all the running services along with their state as well as description.

Step 5 : Inspect a service

systemctl status <service_name>
Content Image

We can also inspect a specific service. Here we can see the state of that service, process id, memory consumption etc.

Step 6 : Startup persistence via services

systemctl list-unit-files --type=service | grep enabled
Content Image

This command tells you the services that persist across reboots and will start automatically when the system boots.

Step 7 : Investigating Cron Jobs

Cron jobs let commands run automatically at specific times, and they survive reboots. There are system-wide tasks, usually for maintenance as well as per user crons for user-specific scheduled tasks.

System cron jobs
cd /etc/cron.d/
Content Image

Content Image

Contains individual cron files for system services and packages. Each file defines its own schedule and command.

We also have /etc/cron.hourly / daily / weekly / monthly. Any script placed here runs automatically at the corresponding interval.

Per-user crontab
crontab -l
Content Image
Content Image

Content Image

Lists cron jobs for the current user.

Lists cron jobs for the root user. Since we are using the root user, we are getting the crons for root. In case, we are logged in with other user sudo crontab -l we have to use sudo for getting crons for root user.

Note : A root-level crontab was created to demonstrate persistence via scheduled tasks. Cron jobs defined in the root crontab execute automatically based on predefined schedules and persist across reboots, making them a common mechanism for administrative automation as well as malicious persistence.

Key log files
Step 1 : Review authentication activity

sudo tail -n 100 /var/log/auth.log
Content Image
Content Image

Here we can see the last 100 lines of the authentication log file. Note that we need to use sudo because /var/log/auth.log is readable only by root. This file contains authentication-related events, such as: - SSH login attempts - Failed passwords - Successful logins - sudo activity - su activity

Step 2: Identify failed event logs

sudo grep -i "Failed" /var/log/auth.log | tail -n 50
Content Image
Here we can see all of the logs related to some kind of failures including : - Failed attempt made by user John to use sudo. Error occured because user John is not in the sudo group - We can also see the failed authentication logs when user John tried to switch to Alice and vice versa. - Since we are filtering the events by the word failed it is also logging us the sudo audit log entry where we can see the commands ran by us that includes the word failed.

Step 3: Identify failed event related to a particular user

sudo grep -i "Failed .* John" /var/log/auth.log | tail -n 50
Content Image
Here we can see the failed login attempts made by user John to root and Alice as well as attempts made by user Alice to John

Step 4: Sudo activity

sudo grep -i "sudo" /var/log/auth.log | tail -n 50
Content Image
sudo grep -i "sudo" /var/log/auth.log | tail -n 50 - It shows the last 50 authentication log entries related to sudo activity.

Specifically, it displays:

When a sudo session was opened
When a sudo session was closed
Which user ran sudo
What command was executed
Step 5: journalctl (systemd logs)

sudo journalctl -u ssh 
Content Image
sudo journalctl - This command displays the entire systemd journal log which includes : System boot messages

Kernel logs
Service start/stop events
Authentication logs
sudo activity
SSH logs
Cron activity
Application logs
sudo journalctl -u ssh" - This will filter out SSH logs from the system logs

Step 6 : Inspect the syslog file

cat /var/log/syslog
Content Image

/var/log/syslog is a general system log file that records: - Service start/stop events - System daemon activity - Network changes - Cron job executions

Step 7: Inspect the kern.log file

cat /var/log/kern.log
Content Image

/var/log/kern.log contains Linux kernel-level events. This file specifically logs:

Kernel boot messages
Hardware detection
Driver initialization
CPU features
Logging through Apache
Step 1 : Locate Apache Log Files

ls -la /var/log/apache2/
Content Image
Here we can see files such as : - access.log : Records all incoming HTTP requests handled by the Apache web server. - error.log : Records server-side errors and warnings. - access.log.1 (Rotated Log) : This is a previous version of access.log. Apache logs are automatically rotated using logrotate to prevent logs from growing indefinitely. - other_vhosts_access.log : Apache log file that records requests for virtual hosts (vhosts) that do not have their own dedicated log files configured. - access.log.bak : This is the backup file for the logs created to demonstrate them for this lab.

Step 2: View Recent Access Log Entries

sudo tail -n 50 /var/log/apache2/access.log.bak
Content Image
GET /admin HTTP/1.1" 404
GET /secret HTTP/1.1" 404
GET /test.php HTTP/1.1" 404
These logs simulates: - Directory probing - Resource enumeration
Repeated 404 responses often indicate reconnaissance activity.

GET /?cmd=whoami HTTP/1.1" 200
This is the log that simulates Command Injection
GET /etc/passwd HTTP/1.1" 404
This generally indicate Directory Traversal Attempt
Step 3: View Recent Error Log Entries

sudo tail -n 20 /var/log/apache2/error.log
Content Image
Here we can see Apache error log recorded both normal service startup events and a permission denial error. The permission error demonstrates how Apache logs server-side access control failures, which are critical for troubleshooting and security monitoring.

Step 4 : Filter and Identify Interesting Events

sudo grep " 404 " /var/log/apache2/access.log | tail -n 20
Content Image
Here we can see the logs with 404 errors(Not found) have been filtered out

Suspicious file indicators
Step 1 : Files without owners

sudo find / -xdev -nouser -o -nogroup 2>/dev/null | head -n 50
Content Image
This usually indicates deleted users or filesystem corruption/tampering.

Step 2 : Executables in writable directories

sudo find /tmp /var/tmp /dev/shm -type f -perm /111 -ls 2>/dev/null
Content Image
This command is used to hunt for suspicious executable files in common temporary and shared memory locations, which are frequently abused for persistence or malware execution.

Note : This file was created and put in the temperoray directory just for the demonstration purpopse, All files in this directory are deleted automatically on reboot. So if you run this command, you wont be seeing any file now.

Step 3 : Recently modified binaries

sudo find /bin /sbin /usr/bin /usr/sbin /usr/local/bin -type f -mtime -15 -ls 2>/dev/null | head -n 50
Content Image

This command is used to identify recently modified system binaries, which is important when checking for persistence, tampering, or unauthorized changes. -m flag in the command is to check for the time period. Here we are checking fir the last 15 days.

Note : When running the lab in the future, try to change the duration to get the modified file.

Step 4 : Unexpected SUID binaries

sudo find / -xdev -perm -4000 -type f -ls 2>/dev/null | head -n 100
This command is used to find SUID (Set User ID) binaries on the system, which are important from a privilege escalation and security perspective. SUID binaries allow normal users to execute certain commands with elevated privileges
Content Image

Step 5 : Find recently modified files in risky locations

sudo find /etc /var /tmp /dev/shm -type f -mtime -3 -ls 2>/dev/null | head -n 100
This command is used to identify recently modified files in critical system and temporary directories, which helps detect configuration changes, persistence artifacts, or suspicious activity.
find /etc /var /tmp /dev/shm Searches important locations: - /etc → system and service configuration files - /var → logs, caches, spool data, cron files - /tmp → temporary files (often abused) - /dev/shm → shared memory (in-memory filesystem, stealthy)

-mtime -3 Finds files modified within the last 3 days.

Content Image

Startup Persistence Inspection
Startup persistence mechanisms allow commands, scripts, or binaries to execute automatically on system boot or user login, enabling long-term access. This section focuses on identifying common user-level and system-level persistence locations.

User-Level Persistence
User-level persistence executes when a specific user logs in or opens a shell.

Step 1: Inspect shell configuration files for suspicious commands or scripts

ls -la /home/<user>/.bashrc /home/<user>/.bash_history /home/<user>/.profile
cat /home/<user>/.bashrc
Content Image
Content Image

.bashrc is a user-level shell configuration file that: - Executes for non-login interactive shells - Runs every time the user opens a new terminal session - Controls environment behavior and shell features

Attackers may modify .bashrc to: - Execute malicious scripts at login - Add reverse shells - Modify PATH for command hijacking

Content Image

.profile is a user login shell initialization file. It executes when: - user logs in via SSH - user logs in via TTY - login shell session starts

.profile is important because: - It executes automatically at login - It can run arbitrary commands

Content Image

.bash_history stores: - Commands executed by the user - In interactive bash sessions

.bash_history is extremely valuable because it: - Shows user intent - Reveals privilege escalation attempts

Step 2 : Search for hidden or suspicious files in Home Directories

ls -la /home/<user>/
find /home/<user> -type f -name ".*"
Content Image
ls -la /home/<user>/ - This command lists all files, including hidden files, in the user’s home directory. The -a flag ensures dot-files are displayed, and -l shows detailed metadata such as permissions, ownership, and timestamps.

find /home/<user> -type f -name ".*" - This command searches recursively for all hidden regular files in the user’s home directory.

In the output what we observed are the standard user configuration files, and no suspicious hidden scripts were identified.

When inspecting hidden files, look for: - Recently created or modified files - Executable hidden files - Hidden scripts referencing /tmp, /dev/shm, or remote URLs

System-Level Persistence
System-level persistence affects the entire system and often runs with elevated privileges.

Step 1 : Inspect system-wide login scripts

cat /etc/profile
ls -la /etc/profile.d/
Content Image

/etc/profile is a system-wide login shell configuration file.
When /etc/profile does run :

SSH login
Console (TTY) login
su - username
First terminal after graphical login
Content Image

/etc/profile.d/ is a directory of scripts that are automatically executed at login.
Any readable .sh file here:

Runs for every user
Runs every login
Step 2 : List enabled system and user services

systemctl list-unit-files --type=service | grep enabled
systemctl --user list-unit-files
Content Image

This command will look for system-wide systemd services which are enabled meaning which are guaranteed to run on every reboot. In this one should llok out for :

Unknown or oddly named services
Recently created or enabled services
Services executing scripts from /tmp, /dev/shm or from hidden directories
Content Image

This systemctl --user list-unit-files command is used to identify user-level systemd services that provide persistence without requiring administrative privileges. These services automatically execute when a user logs in and are commonly abused by attackers to maintain access while avoiding system-level detection.

Attackers prefer this because it:

Blends in with normal desktop services
Survives logouts and reboots
Is harder to correlate with system-level alerts
Inspect SSH authorized keys and SSH configuration
Step 1 : Looking for authorized key for a specific user.

sudo cat /home/<user>/.ssh/authorized_keys 2>/dev/null
Content Image
Here we can see the SSH Authorized Key for user John. Thid will allow user John to authenticate via SSH key-based login

Step 2 : Now let's look at the SSH configuration

sudo cat /etc/ssh/sshd_config
Key settitngs to look at in this file :
Content Image

Here we can see SSH key–based login is enabled. Also we can see it allows SSH login as Root which is insecure since root user has got all the privelages and leak of password can lead to a whole system comprimise.

Safer practice is to allow root login only through SSH keys.

We can see, there are options for setting max auth retries, max sessions per connection, etc.

Content Image

Here we can see the password authentication is enabled for SSH allowing login using username + password.

Conclusion
This lab provided a comprehensive walkthrough of Linux system investigation from a security and monitoring perspective. We examined user and group management, authentication policies, running processes, services, cron jobs, and startup persistence mechanisms at both user and system levels. Key log sources such as auth.log, journalctl, and Apache web logs were analyzed to identify failed logins, privilege escalation attempts, reconnaissance behavior, and server-side errors. Additionally, we reviewed suspicious file indicators, SUID binaries, and SSH configurations to assess potential persistence and misconfigurations. Overall, the lab demonstrates how multiple system artifacts can be correlated to gain visibility into system activity and detect potential security risks.