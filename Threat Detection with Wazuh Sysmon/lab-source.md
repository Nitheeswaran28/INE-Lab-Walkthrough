Solution
Step 1: Open the lab link to access the Attacker (Kali), Ubuntu (Wazuh), and Windows Server 2019 machines.

Attacker Machine

Content Image

Ubuntu Machine

Content Image

Windows Machine

Content Image

Configuring Wazuh Agent
Task 1: Configure Wazuh Agent on Windows Server Machine
First, we need to configure the Wazuh Agent on the Windows Server 2019 so that Wazuh can listen to all the event logs and visualize it in the Wazuh Dashboard.

Before, we run the Wazuh Agent. Switch the Window to Ubuntu Machine and check if the Wazuh is working or not.

Step 2: Running the Firefox Browser and accessing the Wazuh.

URL: https://localhost

[Certificate-warning instruction omitted.]

Username: admin

Password: [redacted]

Content Image

Content Image

We can notice that no active agent is connected with the Wazuh server. Let's switch the view to the Target Machine (Windows) where we have to run the Wazuh Agent.

Let's check the IP address of the Ubuntu Machine.

Command:

ip addr
Content Image

The IP address of the Ubuntu (Wazuh) is: 10.0.26.67

Step 3: The wazuh-agent.msi executable is on the Administrator's Desktop. Run the executable and follow the instruction.

Command:

cd Desktop
msiexec.exe /i wazuh-agent.msi /q WAZUH_MANAGER=10.0.26.67 WAZUH_REGISTRATION_SERVER=10.0.26.67 WAZUH_AGENT_GROUP=default 
Content Image

Wait at least 5-8 minutes to complete the installation

After the installation, start the Wazuh Agent service.

Command:

Start-Service -Name WazuhSvc
Content Image

Note Once you start the service, you might get disconnected from the RDP session. In this case, connect back

The Wazuh Agent service is up and running.

Command:

ipconfig
Content Image

Windows Machine IP address is 10.0.18.168. The same IP address we will use to perform the attacks from the Kali machine.

Let's switch the view to the Ubuntu Machine and access Wazuh Dashboard.

We can observe one active agent after refreshing the page.

Content Image

Task 2: Configure Sysmon on Windows Server

Now, the Windows machine is connected to the Wazuh Manager, reflecting the event logs to the wazuh dashboard. But the logs are coming from the default sources, i.e., Application, Security, etc.

In this case, we want the sensitive logs from sysmon.

What is sysmon?

Sysmon is a Windows system monitoring tool that allows administrators to monitor and log system activity to the Windows event log. It can track a variety of system events, including process creation and termination, network connections, and changes to file creation time. Sysmon also includes a driver component that monitors the operating system at a low level, providing additional information not available through event logging.

Sysmon is developed by Microsoft Sysinternals, and can be used to detect and investigate malicious activity on a Windows system, such as reconnaissance and data exfiltration. Because it logs detailed information about system activity, Sysmon can be used to track down the source of security incidents, such as intrusions and malware infections. Additionally, Sysmon can be used to monitor the behavior of legitimate software and to detect any abnormal activity.

The tool can be configured using a XML configuration file, This allows an administrator to customize what events are logged, as well as specify filters to exclude certain types of activity from being logged. Sysmon supports both 32-bit and 64-bit versions of Windows and it is a command line tool.

Sysmon is a powerful tool for security professionals, but it does produce a high volume of log data, so it is important to have a good logging and event management infrastructure in place when using it. It's also important to note that running sysmon for long time may affect the performance of the system.

Source: https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon

The sysmon executable and its .xml file are provided to you in the C:\Users\Administrator\Documents folder. The .xml file is from SwiftOnSecurity - sysmon-config | A Sysmon configuration file for everybody to fork. This sysmonconfig-export.xml file is the configuration file template with default high-quality event tracing.

Let's start the powershell terminal and run the below command to enable sysmon logs.

Command:*

C:\Users\Administrator\Documents\Sysmon.exe -accepteula -i C:\Users\Administrator\Documents\sysmonconfig-export.xml
Content Image

We have installed sysmon as a service and used the sysmonconfig-export.xml file for high-quality log event tracking.

Learn More About sysmon event logs: https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon

Task 3: Configure Wazuh Agent to send Sysmon log to Wazuh Manager

Now, we want to forward the sysmon log to the Wazuh manager to process and alert based on the given/provided rules.

In this case, we have to edit the ossec.conf file.

The file is in the C:\Program Files (x86)\ossec-agent folder.

Let's open the file using notepad and add the below logfile blog in it.

  <localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>
Also, disable \<client_buffer>. Set value to true
Content Image

Once we save the above configuration and apply these changes, we need to restart the Wazuh Agent.

Command:

Restart-Service -Name WazuhSvc
Content Image

The changes would apply after we restart the service.

Task 4: Create a custom rule to detect running specific executable

Step 4: The Agent is up and working fine. Let's create a custom rule to detect the running specific executable.

In this case, we will detect the running of the mimikatz.exe executable. We can create a custom rule as shown below:

<group name="windows, sysmon, sysmon_process-anomalies,">
   <rule id="100000" level="12">
     <if_group>sysmon_event1</if_group>
     <field name="win.eventdata.image">mimikatz.exe</field>
     <description>Sysmon - Suspicious Process - mimikatz.exe</description>
   </rule>
   <rule id="100001" level="12">
     <if_group>sysmon_event8</if_group>
     <field name="win.eventdata.sourceImage">mimikatz.exe</field>
     <description>Sysmon - Suspicious Process mimikatz.exe created a remote thread</description>
   </rule>
   <rule id="100002" level="12">
     <if_group>sysmon_event_10</if_group>
     <field name="win.eventdata.sourceImage">mimikatz.exe</field>
     <description>Sysmon - Suspicious Process mimikatz.exe accessed $(win.eventdata.targetImage)</description>
   </rule>
</group>
Source: https://wazuh.com/blog/learn-to-detect-threats-on-windows-by-monitoring-sysmon-events/

We can create this rule by editing the local_rules.xml. The file is located in /var/ossec/etc/rules folder.

Command:

nano /var/ossec/etc/rules/local_rules.xml
Before Modification

Content Image

After Modification

Content Image

Once we save the rule, we need to restart the wazuh manager.

Command:

/etc/init.d/wazuh-manager restart
Content Image

After the reboot, the rule would be applied.

[Executable-run instructions omitted.]
Now, again switch back to the Wazuh Dashboard and check if the rule has detected the mimikatz executable.

Go to: Wazuh -> Modules -> Security Events

Content Image

We can notice one alert is present in the dashboard.

Content Image

Content Image

We have successfully detected the mimikatz.exe executable.

Task 5: Allow all sysmon logs to the Wazuh manager

Step 5: Now, currently, we can't see all the sysmon event logs that are very important for identifying the threats. In this case, we will add one more rule showing all the sysmon logs on higher priority.

Command:

<group name="sysmon,">
  <rule id="220000" level="12">
    <if_group>sysmon</if_group>
    <description>Windows Sysmon event. Event ID: $(win.system.eventID)</description>
    <options>no_full_log</options>
  </rule>
</group>
In this case, we also have to edit the same .xml file (local_rules.xml) and add the above code.

Content Image

We need to restart the wazuh manager.

Command:

/etc/init.d/wazuh-manager restart
Content Image

Switch to the Windows and Start Powershell terminal on the Windows Machine

Content Image

Now, again switch back to the Wazuh Dashboard and check if we can see the event ID 1. (Process Creation).

Go to: Wazuh -> Modules -> Security Events

Content Image

Content Image

We have successfully identified all the events. Now all sysmon logs would be present in the Wazuh Dashboard.

Task 6: Perform Pass-The-Hash, Privilege Escalation, and Service Creation attack from the Kali Machine

[Offensive execution procedure and authentication material omitted.]
Detecting the malicious activity on Wazuh Dashboard
The Wazuh dashboard is a web-based interface that allows you to monitor and manage your Wazuh installation. It provides real-time visibility into your security posture, alerts you to potential threats, and allows you to investigate and respond to incidents.

Some specific benefits of using the Wazuh dashboard include:

Real-time visibility: The dashboard provides a centralized view of your security infrastructure, including alerts, events, and logs from your network, servers, and endpoint devices.

Threat detection: Wazuh uses a combination of threat intelligence feeds, behavioral analysis, and rule-based alerts to detect potential threats and vulnerabilities in your environment.

Incident response: The dashboard includes tools for investigating and responding to incidents, such as the ability to search and filter logs, view alerts and events, and review threat intelligence.

Compliance: Wazuh includes features for tracking compliance with security standards and regulations, such as PCI DSS and HIPAA.

Overall, the Wazuh dashboard is a useful tool for organizations looking to improve their security posture and detect and respond to potential threats.

Task 7: Detect the performed attacks on the target machine.

Step 7: We have access to the Wazuh Dashboard using the firefox browser.

Go to: Wazuh -> Modules -> Security Events

Content Image

In this case, we will investigate sysmon events only to detect the threats. Before we examine the sysmon event IDs, let's understand each event ID's meaning.

Events

Event ID 1: Process creation
Event ID 2: A process changed a file creation time
Event ID 3: Network connection
Event ID 4: Sysmon service state changed
Event ID 5: Process terminated
Event ID 6: Driver loaded
Event ID 7: Image loaded
Event ID 8: CreateRemoteThread
Event ID 9: RawAccessRead
Event ID 10: ProcessAccess
Event ID 11: FileCreate
Event ID 12: RegistryEvent (Object create and delete)
Event ID 13: RegistryEvent (Value Set)
Event ID 14: RegistryEvent (Key and Value Rename)
Event ID 15: FileCreateStreamHash
Event ID 16: ServiceConfigurationChange
Event ID 17: PipeEvent (Pipe Created)
Event ID 18: PipeEvent (Pipe Connected)
Event ID 19: WmiEvent (WmiEventFilter activity detected)
Event ID 20: WmiEvent (WmiEventConsumer activity detected)
Event ID 21: WmiEvent (WmiEventConsumerToFilter activity detected)
Event ID 22: DNSEvent (DNS query)
Event ID 23: FileDelete (File Delete archived)
Event ID 24: ClipboardChange (New content in the clipboard)
Event ID 25: ProcessTampering (Process image change)
Event ID 26: FileDeleteDetected (File Delete logged)
Event ID 27: FileBlockExecutable
Event ID 28: FileBlockShredding
Event ID 255: Error
In the future, you may see more events, but till (11-01-2023) these are the events supported by the sysmon.

Read Details about the Event IDs: https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon

We can notice 26 level 12 or above alerts are present.

Note: The logs might be more or less in your case

Content Image

First, let's find out how the PTH attack performed.

While running the Metasploit exploit module on the target machine, the target (payload) was powershell. So, in this case, we will check if any powershell payload has been run on the target machine.

Investigate the Event ID 1; in this event ID, we can discover any newly created process. It could be multiple based on how many new processes has been created.

Content Image

A new service has been created to execute the powershell payload to receive the meterpreter session. The payload is also present in the log. Also, wazuh tagged the event as per the mitre framework event (T1543.003).

Content Image

We can notice that the successful NTLM authentication attempt log is present, and Wazuh has flagged it as a possible pass-the-hash attack.

Also, a service has been created and a registry key has been added for the session. (Event ID 13)

Content Image

Here, we can see the entire powershell payload executed on the target machine to gain the meterpreter session.

Later, we also migrated the current process into the explorer.exe process. That log also should be present in the Wazuh Dashboard.

We can investigate Event ID 8 and observe that we have used powershell to migrate the process from SourceUser: NT AUTHORITY\SYSTEM TargetUser: ATTACKDEFENSE\Administrator".

Content Image

This confirms that process migration happened on the target machine.

We have also created one user, wazuh-hacker, and added it to the localgroup administrators.

Content Image

The user creation and adding that user to the administrator group was also detected successfully.

Investigate the Event ID 1 based on the timestamp, and we can observe the whole command that was executed through the meterpreter session.

Creating the User

Content Image

Adding the user to localgroup Administrator

Content Image

This confirms that user creation activity also happened on the target machine. Similarly, we can provide more good quality logs to the Wazuh dashboard by modifying the target machine's configuration and creating more custom rules to detect faster.