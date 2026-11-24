Solution
Step 1: Open the lab link to access the following machines:

Kali machine

Content Image

Ubuntu Machine

Content Image

Windows Machine

Content Image

Task 1: Deploy the Wazuh Agent on the Windows machine.

Step 2: First, access the Wazuh Dashboard on the Ubuntu machine using the following:

URL: https://localhost
[Certificate-warning instruction omitted.]

Username: admin

Password: [redacted]

Content Image

Content Image

There is currently no active agent connected to the Wazuh server. So now, first let's check the IP address of the Wazuh server machine.

Command:

ip addr
Content Image

The IP address of the Ubuntu machine where Wazuh is running is: 10.0.18.204

Step 3: Next, switch to the Windows machine. The Wazuh agent installer is present under Tools on the Administrator's Desktop. Run the following commands to deploy the Wazuh agent:

Commands:

cd Desktop\Tools
msiexec.exe /i wazuh-agent-4.3.10-1.msi /q WAZUH_MANAGER=10.0.18.204 WAZUH_REGISTRATION_SERVER=10.0.18.204 WAZUH_AGENT_GROUP=default
Content Image

Wait for a few minutes and then start the Wazuh Agent service using the following command:

Command:

Start-Service -Name WazuhSvc
Content Image

Note: Once you start the service, you might get disconnected from the RDP session. In this case, connect back.

Step 4: Now, switch to the Ubuntu machine and refresh the Wazuh Dashboard page till you see an active agent:

Content Image

Task 2: Set up Sysmon and integrate it with Wazuh.

Step 5: On the Windows machine, navigate to the Tools directory present on the Administrator's Desktop. The Sysmon installer is present here. Install Sysmon with the given configuration file using the following command.

Command:

./Sysmon64.exe -accepteula -i sysmonconfig.xml
Content Image

Sysmon is now up and running. You can find the Sysmon events in the Event Viewer by navigating to Applications and Services Logs > Microsoft > Windows > Sysmon.

Content Image

Step 6: Next, configure the agent to capture Sysmon events by adding the following block to the agent configuration file located at C:\Program Files (x86)\ossec-agent\ossec.conf:

<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
Content Image

Restart the agent to apply the changes.

Command:

Restart-Service -Name WazuhSvc
Also, check the IP address of the Windows machine that we will require later.

Command:

ipconfig
Content Image

Windows Machine IP address is 10.0.23.132.

Task 3: Configure a CDB list (constant database) of commonly used ports that you do not want to alert on.

Step 7: A constant database (CDB) list file is a text file that consists of key:value pairs. Each pair must be present on a single line, with the keys being unique. The values are optional. Let's create this file that will contain a list of common ports that we do not want to alert on. On the Ubuntu machine, navigate to the /var/ossec/etc/lists/ directory and save the following content to a file named common-ports.

Commands:

cd /var/ossec/etc/lists/
vi common-ports
File content -

21:
25:
22:
53:
80:
135:
389:
443:
445:
993:
995:
1514:
1515:
3389:
3306:
5000:
5223:
8000:
8002:
8080:
8083:
8443:
Content Image

You can fine tune this list as per your need.

Also, change the ownership and permissions of the file using the following commands:

Commands:

chown wazuh:wazuh common-ports
chmod 660 common-ports
Content Image

Step 8: Next, add this CDB list to the /var/ossec/etc/ossec.conf file within the ruleset block to instruct Wazuh to load the list.

<list>etc/lists/common-ports</list>
Content Image

Restart the manager to apply the changes.

Command:

systemctl restart wazuh-manager
Task 4: Add an appropriate rule to the Wazuh manager.

Step 9: We know that an Event ID 3 is generated on Sysmon when a network connection is detected. Add the following rule to the /var/ossec/etc/rules/local_rules.xml file, to alert when the destination port in the event does not match the entry in our CDB list.

<group name="windows,sysmon,">
    <rule id="115001" level="10">
        <if_sid>61605</if_sid>
        <list field="win.eventdata.destinationPort" lookup="not_match_key">etc/lists/common-ports</list>
        <description>[Network connection]: Network connection to an Uncommon Port $(win.eventdata.destinationPort) by $(win.eventdata.image)</description>
    </rule>
</group>
Content Image

Restart the manager to apply the changes.

Command:

systemctl restart wazuh-manager
Task 5: Simulate an attack (Pass-the-Hash or using a powershell script) and detect the abnormal network connections from the Wazuh Dashboard.

Step 10: First, check the IP address of the Kali machine.

Command:

ifconfig
Content Image

Kali machine's IP address is 10.10.21.2.

[Offensive procedures and credentials omitted. The exercise discusses uncommon destination ports 4444 and 1234. Review matching dashboard alerts as evidence.]

Conclusion
In this lab, we learned how to detect abnormal network connections with Wazuh by generating an alert when the endpoint establishes a network connection to an uncommon port.

References
Wazuh

Sysmon

Detecting Abnormal Network Ports With Wazuh