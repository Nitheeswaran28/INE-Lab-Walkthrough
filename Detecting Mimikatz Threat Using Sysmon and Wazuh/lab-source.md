Solution
Step 1: Open the lab link to access the following machines:

Attacker Machine (Kali)

Content Image

SOC Machine (Wazuh Server)

Content Image

Target Machine (Wazuh Agent)

Content Image

Task 1: Connect Wazuh Agent to the Wazuh Server
First, we need to configure the Wazuh Agent on the Target Machine so that Wazuh can listen to all the event logs and visualize them in the Wazuh Dashboard.

Before we run the Wazuh Agent, check if Wazuh is up and running on the SOC Machine.

Step 2: Access the Wazuh Dashboard using the following:

URL: https://localhost

Username: admin

Password: [redacted]

Content Image

Below is the screenshot of Wazuh Dashboard.

Content Image

There is currently no active agent connected to the Wazuh server. So now, first let's check the IP address of the Wazuh server machine.

Command:

ip addr
Content Image

The IP address of the SOC Machine where Wazuh is running is: 10.0.31.41

Step 3: Next, switch to the Target Machine. Open the C:\Program Files (x86)\ossec-agent\ossec.conf using explorer.

Enter the wazuh server IP i.e. 10.0.31.41.

Content Image

Wait for a few minutes and then start the Wazuh Agent service using the following command on Powershell:

Command:

Start-Service -Name WazuhSvc
Content Image

Step 4: Now, switch to the SOC Machine and refresh the Wazuh Dashboard page till you see an active agent:

Content Image

We have successfully connected the wazuh agent.

Task 2: Configure the Wazuh agent to collect Sysmon logs
Once the Wazuh agent is installed and running on the computer being monitored, we need to set up the agent to monitor Sysmon events.

Step 5: On the Target Machine, configure the agent to capture Sysmon events by adding the following block to the agent configuration file located at C:\Program Files (x86)\ossec-agent\ossec.conf:

Add the following

<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
Content Image

Save and close the file.

Now, we need to configure the agent to keep track of the processes and monitor the logs from Sysmon. In this, we will monitor the Inter-process access, the Process creation, and the Remote thread creation of Mimikatz. This experimental threat is a command-line tool that allows the execution of different operations which may appear suspicious to Sysmon, and therefore, will be registered in the Sysmon section in the Windows event log.

Step 6: Create an XML configuration file named sysconfig.xml in Downloads Folder to process and monitor logs from sysmon.

Reference: https://wazuh.com/blog/learn-to-detect-threats-on-windows-by-monitoring-sysmon-events/

Command:

<Sysmon schemaversion="4.10">
   <HashAlgorithms>md5</HashAlgorithms>
   <EventFiltering>
      <!--SYSMON EVENT ID 1 : PROCESS CREATION-->
      <ProcessCreate onmatch="include">
         <Image condition="contains">mimikatz.exe</Image>
      </ProcessCreate>
      <!--SYSMON EVENT ID 2 : FILE CREATION TIME RETROACTIVELY CHANGED IN THE FILESYSTEM-->
      <FileCreateTime onmatch="include" />
      <!--SYSMON EVENT ID 3 : NETWORK CONNECTION INITIATED-->
      <NetworkConnect onmatch="include" />
      <!--SYSMON EVENT ID 4 : RESERVED FOR SYSMON STATUS MESSAGES, THIS LINE IS INCLUDED FOR DOCUMENTATION PURPOSES ONLY-->
      <!--SYSMON EVENT ID 5 : PROCESS ENDED-->
      <ProcessTerminate onmatch="include" />
      <!--SYSMON EVENT ID 6 : DRIVER LOADED INTO KERNEL-->
      <DriverLoad onmatch="include" />
      <!--SYSMON EVENT ID 7 : DLL (IMAGE) LOADED BY PROCESS-->
      <ImageLoad onmatch="include" />
      <!--SYSMON EVENT ID 8 : REMOTE THREAD CREATED-->
      <CreateRemoteThread onmatch="include">
         <SourceImage condition="contains">mimikatz.exe</SourceImage>
      </CreateRemoteThread>
      <!--SYSMON EVENT ID 9 : RAW DISK ACCESS-->
      <RawAccessRead onmatch="include" />
      <!--SYSMON EVENT ID 10 : INTER-PROCESS ACCESS-->
      <ProcessAccess onmatch="include">
         <SourceImage condition="contains">mimikatz.exe</SourceImage>
      </ProcessAccess>
      <!--SYSMON EVENT ID 11 : FILE CREATED-->
      <FileCreate onmatch="include" />
      <!--SYSMON EVENT ID 12 & 13 & 14 : REGISTRY MODIFICATION-->
      <RegistryEvent onmatch="include" />
      <!--SYSMON EVENT ID 15 : ALTERNATE DATA STREAM CREATED-->
      <FileCreateStreamHash onmatch="include" />
      <PipeEvent onmatch="include" />
   </EventFiltering>
</Sysmon>
Save and close the file.

Content Image

Update and load the configuration file to sysmon.

Command:

cd .\Downloads\Sysmon
.\Sysmon64.exe -c ..\sysconfig.xml
Content Image

Restart the agent to apply changes.

Command:

Restart-Service -Name WazuhSvc
Content Image

We have successfully restarted the wazuh agent to collect sysmon logs.

Task 3: Create rules to detect Mimikatz Attack
Step 6: On the SOC machine(Wazuh Server), add the following detection rules to the /var/ossec/etc/rules/local_rules.xml file.

Add the following

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
Content Image

Save and close the file.

In the above rule configuration:

Rule ID 100000 will allow us to detect Suspicious Process - mimikatz.exe.

Rule ID 100001 will allow us to detect Suspicious Process mimikatz.exe created a remote thread.

Rule ID 100002 detects when the Suspicious Process mimikatz.exe is accessed.

Restart the Wazuh manager to start using the new rules.

Command:

systemctl restart wazuh-manager
Content Image

We have successfully restarted and configured the wazuh manager to start using the new local rules.

Task 4: Review the recorded simulation
[Credential-extraction procedure omitted. The simulation provides process creation, remote thread, and process access detection context.]

Task 5: Detection of Mimikatz Attack with Wazuh
Step 9: Detection

Open Wazuh Dashboard on SOC Machine.

Content Image

Go to Security Event from Wazuh Dashboard.

Content Image

Click on Level 12 or above alerts to filter out severe alert based on our local rules.

Content Image

Alert Generated:
Expand Rule ID 100000 to know more detailed information
Content Image

Expand Rule ID 100002 to know more detailed information
Content Image

This is all for the lab!

Conclusion
In this lab, we have demonstrated how to use Mimikatz threat tool to experiment with Windows security to monitor and report each event of interest in the EventChannel log format. We have also learnt that monitoring the Windows event log with Wazuh is as simple as configuring the agent to monitor any desired channel such as the Sysmon channel.

References
Learn to detect threats on Windows by monitoring Sysmon events
Wazuh
Mimikatz threat against Windows security