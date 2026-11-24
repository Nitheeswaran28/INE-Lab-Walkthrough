Step 1: Launch the lab to access the following 3 machines:

SOC Machine hosting Splunk:

Content Image

Ubuntu Machine to be monitored:

Content Image

Windows Machine to be monitored:

Content Image

Step 2: Verify that the Splunk server is running on the SOC Machine. Open firefox and navigate to http://localhost:8000.

You should see the Splunk Web dashboard.

Content Image

Identify the IP address of the Splunk server machine using the following command:

ip addr
Content Image

The IP address of the Splunk server machine is 10.0.21.229. This may vary for you. Replace wherever <SPLUNK_IP> appears in this guide with this IP address.

Part A: Configure the Ubuntu Machine
Enable Event Logging on Ubuntu
Step 3: On the Ubuntu Machine, verify that rsyslog is installed and running. Open a terminal and run:

systemctl status rsyslog
Content Image

rsyslog is the default system logging daemon on Ubuntu. It should be active and running. If it is not running, start it with:

systemctl start rsyslog
systemctl enable rsyslog
Step 4: Verify that key log files are being generated. Check for the existence of important log files:

ls -la /var/log/syslog
ls -la /var/log/auth.log
ls -la /var/log/kern.log
Content Image

Step 5: Enable audit logging for more detailed security event tracking. The auditd (Linux Audit Daemon) package is pre-installed on this machine. Start and enable it:

systemctl start auditd
systemctl enable auditd
Content Image

Verify that auditd is active and running:

systemctl status auditd
Content Image

Step 6: Verify that the audit log file exists:

ls -la /var/log/audit/audit.log
Content Image

These are the primary log files we will forward to Splunk:

/var/log/syslog — General system activity logs
/var/log/auth.log — Authentication and authorization logs (SSH logins, sudo usage, etc.)
/var/log/kern.log — Kernel messages
/var/log/audit/audit.log — Detailed security audit events (file access, system calls, user actions)
Install and Configure Splunk Universal Forwarder on Ubuntu
Step 7: The Splunk Universal Forwarder installer has been pre-placed on the Desktop folder of the system. Install it using dpkg:

cd Desktop
dpkg -i splunkforwarder-linux-amd64.deb
Content Image

The forwarder will be installed to /opt/splunkforwarder.

Step 8: Start the Splunk Universal Forwarder for the first time and accept the license agreement:

/opt/splunkforwarder/bin/splunk start --accept-license
You will be prompted to create an administrator username and password for the forwarder. Enter your desired credentials. (For example, admin:pass1234)

Content Image

Step 9: Configure the forwarder to send data to the Splunk server (SOC Machine) at <SPLUNK_IP> on the default receiving port 9997:

/opt/splunkforwarder/bin/splunk add forward-server <SPLUNK_IP>:9997
Enter the forwarder admin credentials when prompted.

Content Image

Step 10: Add the log files to be monitored by the forwarder. We will monitor syslog, auth.log, kern.log, and the audit.log:

/opt/splunkforwarder/bin/splunk add monitor /var/log/syslog -index main
/opt/splunkforwarder/bin/splunk add monitor /var/log/auth.log -index main
/opt/splunkforwarder/bin/splunk add monitor /var/log/kern.log -index main
/opt/splunkforwarder/bin/splunk add monitor /var/log/audit/audit.log -index main
Content Image

Step 11: The Splunk Forwarder identifies files by computing a CRC checksum of their first 256 bytes. Since Linux log files like syslog, auth.log, and kern.log often share overlapping content (e.g., the same boot messages), Splunk may compute identical CRCs and mistakenly treat them as the same file — causing some logs (particularly syslog) to be silently skipped.

To fix this, edit the forwarder's inputs.conf to add crcSalt = <SOURCE> to the syslog monitor. This tells Splunk to include the file path in the CRC calculation, ensuring each file gets a unique identifier:

nano /opt/splunkforwarder/etc/apps/search/local/inputs.conf
Add crcSalt = <SOURCE> under the [monitor:///var/log/syslog] stanza so it looks like this:

[monitor:///var/log/syslog]
disabled = false
index = main
crcSalt = <SOURCE>
Content Image

Save the file and exit the editor.

Step 12: Restart the Splunk Forwarder to apply all configuration changes:

/opt/splunkforwarder/bin/splunk restart
Content Image

Configure the Splunk Server to Receive Data
Step 13: On the SOC Machine, configure Splunk to listen for incoming data from forwarders. Navigate to Settings > Forwarding and receiving.

Content Image

Step 14: Under Receive data, click Configure receiving.

Content Image

Step 15: Click New Receiving Port. Enter port 9997 and click Save.

Content Image

The receiving port is now configured and Splunk will accept data from forwarders on port 9997.

Content Image

Generate Test Events on Ubuntu
Step 16: On the Ubuntu Machine, generate some test events to verify that log forwarding is working. Run the following commands to produce authentication and system log entries:

Generate a failed SSH login attempt (use a wrong password when prompted):

ssh invaliduser@localhost
Content Image

Generate sudo events:

sudo ls /root
Content Image

Generate a custom syslog message using the logger command:

logger "Splunk Test Event: This is a test log entry from Ubuntu Machine"
Content Image

Step 17: Verify that the test events were written to the local log files:

tail -5 /var/log/auth.log
tail -5 /var/log/syslog
Content Image

Verify Ubuntu Logs in Splunk Dashboard
Step 18: Go back to the SOC Machine and open the Splunk Web interface. Navigate to Search & Reporting from the left-hand panel or the main dashboard.

Content Image

Step 19: In the search bar, search for all events from the Ubuntu Machine by running the following SPL (Search Processing Language) query:

index=main host=<ubuntu-hostname>
Replace <ubuntu-hostname> with the actual hostname of the Ubuntu Machine. In this case, it is ip-10-0-28-202.

Click on Search.

Content Image

Step 20: Verify that the test events appear in the search results. Search specifically for the messages:

index=main host=ip-10-0-28-202 "invaliduser"
index=main host=ip-10-0-28-202 "ls /root"
index=main host=ip-10-0-28-202 "Splunk Test Event"
Content Image

Content Image

Content Image

This confirms that:

Event logging is enabled on the Ubuntu Machine
The Splunk Universal Forwarder is correctly installed and configured
Logs are being forwarded to the Splunk server
The Splunk server is receiving and indexing the forwarded logs
Part B: Configure the Windows Machine
Enable Event Logging on Windows
Step 21: On the Windows Machine, open Event Viewer. Press Win + R, type eventvwr.msc, and press Enter.

Content Image

Step 22: In Event Viewer, expand Windows Logs in the left panel. Verify that the following logs are present and actively recording events:

Application — Logs from applications and programs
Security — Security-related events such as logon attempts
System — Windows system component events
Setup — Events related to application setup
Content Image

Step 23: Enable audit policies for more granular security logging. Open Local Security Policy by pressing Win + R, typing secpol.msc, and pressing Enter.

Content Image

Step 24: Navigate to Security Settings > Local Policies > Audit Policy. Enable auditing for the following categories by setting both Success and Failure:

Audit account logon events
Audit account management
Audit logon events
Audit process tracking
Audit system events
Double-click each policy, check both Success and Failure, and click OK.

Content Image

Content Image

Step 25: Enable PowerShell Script Block Logging for enhanced visibility into PowerShell activity. Open Group Policy Editor by pressing Win + R, typing gpedit.msc, and pressing Enter.

Navigate to Computer Configuration > Administrative Templates > Windows Components > Windows PowerShell.

Double-click Turn on PowerShell Script Block Logging, select Enabled, and click OK.

Content Image

Content Image

Install and Configure Splunk Universal Forwarder on Windows
Step 26: The Splunk Universal Forwarder installer for Windows has been pre-placed on the Desktop folder of the system. Double-click the .msi file to launch the installer.

Content Image

Step 27: The Splunk Universal Forwarder Setup Wizard will open. Check the box to accept the License Agreement and click Next.

Content Image

Step 28: Enter a desired credentials for the forwarder (for example, admin:pass1234), then click Next.

Content Image

Step 29: When prompted for the Deployment Server, you may leave this blank and click Next (we are not using a deployment server in this lab).

Content Image

Step 30: On the Receiving Indexer screen, enter the Splunk server details:

Hostname or IP: <SPLUNK_IP>
Port: 9997
Click Next.

Content Image

Step 31: Click Install to begin the installation.

Content Image

Wait for the installation to complete, then click Finish.

Content Image

Step 32: Open a Command Prompt. Configure the Windows Event Logs to be monitored by the forwarder.

Create the inputs.conf using Notepad:

notepad "C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf"
Add the following content to monitor the Security, System, Application, and PowerShell event logs:

[WinEventLog://Security]
index = main
disabled = false
renderXml = false

[WinEventLog://System]
index = main
disabled = false
renderXml = false

[WinEventLog://Application]
index = main
disabled = false
renderXml = false

[WinEventLog://Microsoft-Windows-PowerShell/Operational]
index = main
disabled = false
renderXml = false
Save the file and close Notepad.

Content Image

Step 33: Navigate to the Splunk Forwarder installation directory and verify the forwarder configuration by checking the forward-server list:

cd "C:\Program Files\SplunkUniversalForwarder\bin"
splunk list forward-server
Enter credentials when prompted.

Content Image

Step 34: Restart the Splunk Universal Forwarder to apply all changes:

splunk restart
Content Image

Generate Test Events on Windows
Step 35: Generate test events on the Windows Machine to verify log forwarding. Open Command Prompt and generate a failed login event:

runas /user:fakeuser cmd
Enter any password when prompted. This will fail and generate a Security event log entry (Event ID 4625 — Failed logon attempt).

Content Image

Step 36: Generate a Windows System event by restarting a service:

net stop spooler
net start spooler
Content Image

Step 37: Generate a PowerShell event by opening PowerShell and running a test script:

Write-EventLog -LogName Application -Source "Application" -EventId 9999 -EntryType Information -Message "Splunk Test Event: This is a test log entry from Windows Machine"
Content Image

Verify Windows Logs in Splunk Dashboard
Step 38: Go back to the SOC Machine and open the Splunk Web interface. Navigate to Search & Reporting.

Content Image

Step 39: Search for events from the Windows Machine by running the following SPL query:

index=main host=<windows-hostname>
Replace <windows-hostname> with the actual hostname of the Windows Machine. In this case, it is "ATTACKDEFENSE"

Content Image

Step 40: Search specifically for the test event generated using PowerShell:

index=main "Splunk Test Event" host="ATTACKDEFENSE"
Content Image

Step 41: Search for events related to Spooler service:

index=main host="ATTACKDEFENSE" *spooler*
Content Image

Step 42: Search for failed logon events from the Windows Machine:

index=main EventCode=4625 host="ATTACKDEFENSE" "fakeuser"
Content Image

This confirms that:

Event logging is enabled on the Windows Machine
The Splunk Universal Forwarder is correctly installed and configured
Logs are being forwarded to the Splunk server
The Splunk server is receiving and indexing the forwarded logs
Part C: Exploring the Splunk Dashboard
Step 43: On the Splunk Web interface, navigate to Search & Reporting. Run a broad search to see all events from both machines:

index=main
Content Image

Step 44: Use the Fields sidebar on the left to explore the ingested data. Click on host to see a breakdown of events by host. You should see both the Ubuntu and Windows machine hostnames listed.

Content Image

Step 45: Click on source in the Fields sidebar to see the different types of logs being collected (e.g., /var/log/syslog, /var/log/auth.log, WinEventLog:Security, WinEventLog:System etc.).

Content Image

Step 46: Create a simple visualization. Run the following query to see a time chart of events per host:

index=main | timechart count by host
Content Image

Click on Visualization tab to see the data as a chart.

Content Image

Step 47: Run a query to see the top event sources:

index=main | top source
Content Image

This provides a summary of which types of logs are generating the most events.

Step 48: Run a security-focused query to look for all authentication-related events across both machines:

index=main (source="/var/log/auth.log" OR source="WinEventLog:Security")
Content Image

This demonstrates Splunk's ability to aggregate and search across security logs from multiple hosts and operating systems in a single query.

Conclusion
In this lab, you have successfully:

Enabled event logging on both Ubuntu (rsyslog, auditd) and Windows (Event Viewer, Audit Policies, PowerShell logging) machines
Installed and configured the Splunk Universal Forwarder on both Ubuntu (.deb package) and Windows (.msi installer) to forward logs to the Splunk server
Configured the Splunk server to receive data on port 9997
Generated test events on both machines to verify the logging and forwarding pipeline
Verified log ingestion in the Splunk dashboard using SPL queries
Explored the Splunk dashboard by creating searches, visualizations, and cross-platform security queries