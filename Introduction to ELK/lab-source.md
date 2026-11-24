Step 1: Launch the lab to access the following 3 machines:

SOC Machine hosting ELK Stack:

Content Image

Ubuntu Machine to be monitored:

Content Image

Windows Machine to be monitored:

Content Image

Step 2: Verify that the ELK Stack is running on the SOC Machine. Open a terminal and check that Elasticsearch is active:

systemctl status elasticsearch
Content Image

Verify that Kibana is running:

systemctl status kibana
Content Image

Open Firefox and navigate to http://localhost:5601. You should see the Kibana Web interface.

Content Image

Identify the IP address of the ELK server machine using the following command:

ip addr
Content Image

Note the IP address of the ELK server machine. This may vary for you. Replace wherever <ELK_IP> appears in this guide with this IP address.

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

These are the primary log files we will forward to Elasticsearch:

/var/log/syslog — General system activity logs
/var/log/auth.log — Authentication and authorization logs (SSH logins, sudo usage, etc.)
/var/log/kern.log — Kernel messages
/var/log/audit/audit.log — Detailed security audit events (file access, system calls, user actions)
Install and Configure Filebeat on Ubuntu
Step 7: The Filebeat installer has been pre-placed on the Desktop folder of the system. Install it using dpkg:

cd Desktop
dpkg -i filebeat-9.0.0-amd64.deb
Content Image

Filebeat will be installed to /etc/filebeat/ (configuration) and /usr/share/filebeat/ (binaries).

Step 8: Configure Filebeat to collect the required log files and send them to Elasticsearch. Edit the Filebeat configuration file:

nano /etc/filebeat/filebeat.yml
Locate the filebeat.inputs section. You will see a default filestream input. Comment out the defaults. Add inputs to monitor the required log files as following:

filebeat.inputs:
  - type: filestream
    id: syslog
    enabled: true
    paths:
      - /var/log/syslog

    - type: filestream
    id: authlog
    enabled: true
    paths:
      - /var/log/auth.log

    - type: filestream
    id: kernlog
    enabled: true
    paths:
      - /var/log/kern.log

    - type: filestream
    id: auditlog
    enabled: true
    paths:
      - /var/log/audit/audit.log
Content Image

Step 9: In the same filebeat.yml file, locate the output.elasticsearch section. Configure it to point to the ELK server:

output.elasticsearch:
  hosts: ["<ELK_IP>:9200"]
Make sure the output.logstash section is commented out (lines prefixed with #), as we are sending data directly to Elasticsearch.

Content Image

Step 10: Optionally, configure the Kibana endpoint in filebeat.yml to enable dashboard setup. Locate the setup.kibana section and set:

setup.kibana:
  host: "<ELK_IP>:5601"
Content Image

Save the file and exit the editor.

Step 11: Test the Filebeat configuration to ensure there are no syntax errors:

filebeat test config
Content Image

Test the connection to Elasticsearch:

filebeat test output
Content Image

Both tests should return successful results.

Step 12: Start and enable the Filebeat service:

systemctl start filebeat
systemctl enable filebeat
Content Image

Verify that Filebeat is running:

systemctl status filebeat
Content Image

Generate Test Events on Ubuntu
Step 13: On the Ubuntu Machine, generate some test events to verify that log forwarding is working. Run the following commands to produce authentication and system log entries:

Generate a failed SSH login attempt (use a wrong password when prompted):

ssh invaliduser@localhost
Content Image

Generate sudo events:

sudo ls /root
Content Image

Generate a custom syslog message using the logger command:

logger "ELK Test Event: This is a test log entry from Ubuntu Machine"
Content Image

Step 14: Verify that the test events were written to the local log files:

tail -5 /var/log/auth.log
tail -5 /var/log/syslog
Content Image

Content Image

Verify Ubuntu Logs in Kibana Dashboard
Step 15: Go back to the SOC Machine and open the Kibana Web interface at http://localhost:5601.

First, we need to create an index pattern so Kibana can discover the Filebeat data. Navigate to Management (gear icon) > Stack Management > Index Patterns (under Kibana section).

Content Image

Content Image

Step 16: Click Create index pattern. In the Name field, enter filebeat-*.

Content Image

Step 17: Select @timestamp as the Time field from the dropdown, then click Create index pattern.

Content Image

The index pattern is now created and Kibana can search across Filebeat data.

Content Image

Step 18: Navigate to Discover from the left-hand menu (compass icon). Make sure the filebeat-* index pattern is selected in the dropdown.

Content Image

Step 19: In the search bar, search for events from the Ubuntu Machine by running the following KQL (Kibana Query Language) query:

host.name: "<ubuntu-hostname>"
Replace <ubuntu-hostname> with the actual hostname of the Ubuntu Machine.

Content Image

Step 20: Verify that the test events appear in the search results. Search specifically for the test messages:

host.name: "ip-10-0-20-21" and message: "invaliduser"
host.name: "ip-10-0-20-21" and message: "ls /root"
host.name: "ip-10-0-20-21" and message: "ELK Test Event"
Content Image

Content Image

Content Image

This confirms that:

Event logging is enabled on the Ubuntu Machine
Filebeat is correctly installed and configured
Logs are being forwarded to Elasticsearch
The ELK server is receiving and indexing the forwarded logs
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

Step 25: Enable PowerShell Script Block Logging for enhanced visibility into PowerShell activity. Open Group Policy Editor by pressing Win + R, typing gpedit.msc, and pressing Enter.

Navigate to Computer Configuration > Administrative Templates > Windows Components > Windows PowerShell.

Double-click Turn on PowerShell Script Block Logging, select Enabled, and click OK.

Content Image

Content Image

Install and Configure Winlogbeat on Windows
Step 26: The Winlogbeat installer for Windows has been pre-placed on the Desktop folder of the system. Extract the winlogbeat-9.0.0-windows-x86_64.zip archive to C:\Program Files\Winlogbeat.

Right-click the zip file and select Extract All. Set the destination to C:\Program Files\Winlogbeat.

Content Image

Step 27: Open a PowerShell window as Administrator. Navigate to the Winlogbeat directory:

cd "C:\Program Files\Winlogbeat\winlogbeat-9.0.0-windows-x86_64"
Content Image

Step 28: Edit the Winlogbeat configuration file winlogbeat.yml using Notepad:

notepad winlogbeat.yml
Locate the winlogbeat.event_logs section and configure it to monitor the required event logs. Comment out the defaults.

winlogbeat.event_logs:
  - name: Application
    - name: Security
    - name: System
    - name: Microsoft-Windows-PowerShell/Operational
Content Image

Step 29: In the same winlogbeat.yml file, locate the output.elasticsearch section. Configure it to point to the ELK server:

output.elasticsearch:
  hosts: ["<ELK_IP>:9200"]
Make sure the output.logstash section is commented out (lines prefixed with #).

Content Image

Step 30: Optionally, configure the Kibana endpoint for dashboard setup:

setup.kibana:
  host: "<ELK_IP>:5601"
Save the file and close Notepad.

Content Image

Step 31: Test the Winlogbeat configuration to ensure there are no errors:

.\winlogbeat.exe test config
Content Image

Test the connection to Elasticsearch:

.\winlogbeat.exe test output
Content Image

Step 32: Install Winlogbeat as a Windows service by running the installation script:

.\install-service-winlogbeat.ps1
If prompted about execution policy, run the following command first and then re-run the install script:

Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope Process
Content Image

Step 33: Start the Winlogbeat service:

Start-Service winlogbeat
Verify the service is running:

Get-Service winlogbeat
Content Image

Generate Test Events on Windows
Step 34: Generate test events on the Windows Machine to verify log forwarding. Open Command Prompt and generate a failed login event:

runas /user:fakeuser cmd
Enter any password when prompted. This will fail and generate a Security event log entry (Event ID 4625 — Failed logon attempt).

Content Image

Step 35: Generate a Windows System event by restarting a service:

net stop spooler
net start spooler
Content Image

Step 36: Generate a PowerShell event by opening PowerShell and running a test script:

Write-EventLog -LogName Application -Source "Application" -EventId 9999 -EntryType Information -Message "ELK Test Event: This is a test log entry from Windows Machine"
Content Image

Verify Windows Logs in Kibana Dashboard
Step 37: Go back to the SOC Machine and open the Kibana Web interface. Navigate to Management > Stack Management > Index Patterns.

Create a new index pattern for Winlogbeat data. Click Create index pattern, enter winlogbeat-* as the name.

Content Image

Step 38: Select @timestamp as the Time field and click Create index pattern.

Content Image

Step 39: Navigate to Discover. Select the winlogbeat-* index pattern from the dropdown. Search for events from the Windows Machine:

host.name: "<windows-hostname>"
Replace <windows-hostname> with the actual hostname of the Windows Machine (e.g., ATTACKDEFENSE).

Content Image

Step 40: Search specifically for the test event generated using PowerShell:

host.name: "attackdefense" and message: "ELK Test Event"
Content Image

Step 41: Search for events related to the Spooler service:

host.name: "attackdefense" and message: "spooler"
Content Image

Step 42: Search for failed logon events from the Windows Machine:

host.name: "attackdefense" and winlog.event_id: "4625" and message: "fakeuser"
Content Image

This confirms that:

Event logging is enabled on the Windows Machine
Winlogbeat is correctly installed and configured
Logs are being forwarded to Elasticsearch
The ELK server is receiving and indexing the forwarded logs
Part C: Exploring the Kibana Dashboard
Step 43: On the Kibana Web interface, navigate to Discover. To see all events from both machines, you can create a combined index pattern. Navigate to Management > Stack Management > Index Patterns and create a new pattern with the name *beat-* to match both filebeat-* and winlogbeat-* indices. Select @timestamp as the time field.

Content Image

Step 44: In Discover, select the *beat-* index pattern. Use the Available fields panel on the left to explore the ingested data. Click on host.name to see a breakdown of events by host. You should see both the Ubuntu and Windows machine hostnames listed.

Content Image

Step 45: Click on log.file.path (for Filebeat) and winlog.channel (for Winlogbeat) in the Available fields panel to see the different types of logs being collected (e.g., /var/log/syslog, /var/log/auth.log, Security, System, etc.).

Content Image

Step 46: Create a simple visualization. Navigate to Visualize Library (chart icon in the left menu) and click Create visualization. Select Lens as the visualization type.

Content Image

Content Image

Make sure the *beat-* index pattern in selected.

Create a time-based bar chart: - Drag @timestamp to the horizontal axis - For the vertical axis, select Count of records - Drag host.name to the Break down by field

This will show a time chart of events per host.

Content Image

Click Save to save the visualization.

Step 47: Run a security-focused query in Discover to look for all authentication-related events across both machines. Select the *beat-* index pattern and run:

log.file.path: "/var/log/auth.log" or winlog.channel: "Security"
Content Image

This demonstrates the ELK Stack's ability to aggregate and search across security logs from multiple hosts and operating systems in a single query.

Conclusion
In this lab, you have successfully:

Enabled event logging on both Ubuntu (rsyslog, auditd) and Windows (Event Viewer, Audit Policies, PowerShell logging) machines
Installed and configured Filebeat on Ubuntu to forward logs (syslog, auth.log, kern.log, audit.log) to Elasticsearch
Installed and configured Winlogbeat on Windows to forward Windows Event Logs (Security, System, Application, PowerShell) to Elasticsearch
Generated test events on both machines to verify the logging and forwarding pipeline
Verified log ingestion in the Kibana dashboard using KQL queries
Explored the Kibana dashboard by creating index patterns, searching with Discover, building visualizations, and running cross-platform security queries
