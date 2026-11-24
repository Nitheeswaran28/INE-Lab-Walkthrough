Step 1: Open the lab link to access the following machines:

Attacker Machine (Kali)

[Source image reference omitted.]

SOC Machine (Wazuh Server)

[Source image reference omitted.]

Target Machine (Wazuh Agent)

[Source image reference omitted.]

First, we need to configure the Wazuh Agent on the Target Machine so that Wazuh can listen to all the event logs and visualize them in the Wazuh Dashboard.

Before we run the Wazuh Agent, check if Wazuh is up and running on the SOC Machine.

Step 2: Access the Wazuh Dashboard using the following:
- URL: https://localhost
- Username: admin
- Password: [redacted]

[Source image reference omitted.]

Below is the screenshot of Wazuh Dashboard.

[Source image reference omitted.]

There is currently no active agent connected to the Wazuh server. So now, first let's check the IP address of the Wazuh server machine.

Command:
```
ip addr
```

[Source image reference omitted.]

The IP address of the SOC Machine where Wazuh is running is: 10.0.28.182

Step 3: Next, switch to the Target Machine. Open the Wazuh Agent Manager from the Taskbar.

Enter the wazuh server IP i.e. 10.0.28.182.

[Source image reference omitted.]

Wait for a few minutes and then start the Wazuh Agent service using the following command on Powershell:

Command:
```
Start-Service -Name WazuhSvc
```

[Source image reference omitted.]

Step 4: Now, switch to the SOC Machine and refresh the Wazuh Dashboard page till you see an active agent:

[Source image reference omitted.]

We have successfully connected the wazuh agent.
We can detect PsExec usage attack activities by enriching the victim endpoint logs with Sysmon. These logs are then analyzed against a custom ruleset defined on the Wazuh server.

Step 5: On the Target Machine, configure the agent to capture Sysmon events by adding the following block to the agent configuration file located at C:\Program Files (x86)\ossec-agent\ossec.conf:

Add the following
```
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

[Source image reference omitted.]

Save and close the file.

Open PowerShell to install and configure Sysmon using the command below:

Reference: https://wazuh.com/resources/blog/emulation-of-attack-techniques-and-detection-with-wazuh/sysmonconfig.xml

Command:
```
cd .\Downloads\Sysmon
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

[Source image reference omitted.]

As you can verify the sysmon has started successfully.

Restart the agent to apply changes.

Command:
```
Restart-Service -Name WazuhSvc
```

[Source image reference omitted.]

We have successfully restarted the wazuh agent to collect sysmon logs.
Step 6: On the SOC machine(Wazuh Server), add the following detection rules to the /var/ossec/etc/rules/local_rules.xml file.

Add the following
```
<group name="windows,sysmon">
  <rule id="160010" level="10">
    <if_group>windows,sysmon</if_group> 
    <field name="win.eventdata.eventType" type="pcre2" >SetValue</field>
    <field name="win.eventdata.targetObject" type="pcre2" >\\\\SOFTWARE\\\\Sysinternals\\\\PsExec\\\\EulaAccepted</field>
    <options>no_full_log</options>
    <description>PsExec EULA registry found on $(win.system.computer)</description>
      <mitre>
        <id>T1112</id>
      </mitre>
  </rule>

  <rule id="160011" level="10">
    <if_group>windows,sysmon</if_group> 
    <field name="win.eventdata.targetObject" type="pcre2" >HKLM\\\\System\\\\CurrentControlSet\\\\Services\\\\PSEXESVC</field>
    <field name="win.eventdata.eventType" type="pcre2" >^SetValue$</field>
    <field name="win.eventdata.user" type="pcre2" >NT AUTHORITY\\\\SYSTEM</field>
    <options>no_full_log</options> 
    <description>PsExec service running as $(win.eventdata.user) has been created on $(win.system.computer)</description>
      <mitre>
        <id>T1543.003</id>
      </mitre>
  </rule>  

  <rule id="160012" level="10">
    <if_group>windows,sysmon</if_group> 
    <field name="win.eventdata.targetObject" type="pcre2" >HKLM\\\\System\\\\CurrentControlSet\\\\Services\\\\PSEXESVC</field>
    <field name="win.eventdata.eventType" type="pcre2" >^DeleteKey$</field>
    <field name="win.eventdata.user" type="pcre2" >NT AUTHORITY\\\\SYSTEM</field>
    <options>no_full_log</options>
    <description>PsExec service running as $(win.eventdata.user) has been deleted on $(win.system.computer)</description>
      <mitre>
        <id>T1543.003</id>
      </mitre>
  </rule>

  <rule id="160013" level="10">
    <if_group>windows,sysmon</if_group> 
    <field name="win.eventdata.eventType" type="pcre2" >CreatePipe</field>
    <field name="win.eventdata.image" type="pcre2" >\\\\PSEXESVC.exe</field>
    <match type="pcre2">(?i)(PSEXESVC|stdin|stdout|stderr)</match>
    <field name="win.eventdata.user" type="pcre2" >NT AUTHORITY\\\\SYSTEM</field>
    <options>no_full_log</options> 
    <description>PsExec named pipe running as $(win.eventdata.user) has been found on $(win.system.computer)</description>
      <mitre>
        <id>T1570</id>
      </mitre>
  </rule>
</group>
```

[Source image reference omitted.]

Save and close the file.

T1543.003: Create or Modify System Process: Windows Service

Adversaries may create or modify Windows services to repeatedly execute malicious payloads as part of persistence. When Windows boots up, it starts programs or applications called services that perform background system functions. Windows service configuration information, including the file path to the service's executable or recovery programs/commands, is stored in the Windows Registry.

Adversaries may install a new service or modify an existing service to execute at startup to persist on a system. Service configurations can be set or modified using system utilities (such as sc.exe), by directly modifying the Registry, or by interacting directly with the Windows API.

Adversaries may also use services to install and execute malicious drivers. For example, after dropping a driver file (ex: .sys) to disk, the payload can be loaded and registered via Native API functions such as CreateServiceW() (or manually via functions such as ZwLoadDriver() and ZwSetValueKey()), by creating the required service Registry values (i.e. Modify Registry), or by using command-line utilities such as PnPUtil.exe. Adversaries may leverage these drivers as Rootkits to hide the presence of malicious activity on a system. Adversaries may also load a signed yet vulnerable driver onto a compromised machine (known as "Bring Your Vulnerable Driver" (BYOVD)) as part of Exploitation for Privilege Escalation.

Services may be created with administrator privileges but are executed under SYSTEM privileges, so an adversary may also use a service to escalate privileges. Adversaries may also directly start services through Service Execution. To make detection analysis more challenging, malicious services may also incorporate Masquerade Task or Service (ex: using a service and/or payload name related to a legitimate OS or benign software component).

Reference: https://attack.mitre.org/techniques/T1543/003/

In the above rule configuration:
- Rule ID 160010 will allow us to detect the PsExec EULA registry.
Reference: https://attack.mitre.org/techniques/T1112
- Rule ID 160011 will allow us to detect newly created PsExec service running as NT AUTHORITY\SYSTEM.
Reference: https://attack.mitre.org/techniques/T1543/003/
- Rule ID 160012 detects when the PsExec service running as NT AUTHORITY\SYSTEM has been deleted on the monitored system.
Reference: https://attack.mitre.org/techniques/T1543/003
- Rule ID 160013 detects when the PsExec named pipe associated with stdin, stdout, or stderr is running as NT AUTHORITY\SYSTEM on the monitored system.
Reference: https://attack.mitre.org/techniques/T1570

Restart the Wazuh manager to start using the new rules.

Command:
```
systemctl restart wazuh-manager
```

[Source image reference omitted.]

We have successfully restarted and configured the wazuh manager to start using the new local rules.
Step 7: Review recorded PsExec activity
[Privilege-escalation procedure omitted.] The notes report that the resulting PowerShell process ran as NT AUTHORITY\SYSTEM.

Step 8: Detection

Open Wazuh Dashboard on SOC Machine.

[Source image reference omitted.]

Go to Security Event from Wazuh Dashboard.

[Source image reference omitted.]

Scroll down and search for rule IDs which were specified in the local rule files.

[Source image reference omitted.]

- Expand Rule ID 160011 to know more detailed information

[Source image reference omitted.]

- Expand Rule ID 160012 to know more detailed information

[Source image reference omitted.]

- Expand Rule ID 160013 to know more detailed information

[Source image reference omitted.]

This is all for the lab!
In this lab, we have demonstrated how to detect the usage of PsExec on the Windows endpoint. PsExec service creation was detected out of the box by Wazuh, while other activities were detected on the Wazuh platform using custom Wazuh detection rules.