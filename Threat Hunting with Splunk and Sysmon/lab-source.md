Solution
Step 1: Open the lab link to access the following machines:

Ubuntu Machine

Content Image

Windows Machine

Content Image

Task 1: Set up and configure Splunk (free license) on the Ubuntu machine.

Step 2: First, go to /root/Desktop/Tools where the Splunk installer file is present. Use it to install Splunk. This might take a few minutes.

Command:

dpkg -i splunk-9.0.3-dd0128b1f8cd-linux-2.6-amd64.deb
Content Image

Step 3: Now, navigate to /opt/splunk/bin and then run the following command to start Splunk while accepting the license.

Command:

./splunk start --accept-license
You will be prompted to set the administrator username and password. You can set the credentials as follows:

Username: admin
Password: [redacted]
Content Image

Ignore any error that you might notice. Splunk is now up and running.

Content Image

To enable Splunk to start automatically at boot time, you can use the following command:

Command:

./splunk enable boot-start
Content Image

Step 4: Navigate to the following URL to access Splunk:

http://localhost:8000
Use the credentials that you specified before to log in:

Username: admin
Password: [redacted]
Content Image

Step 5: Once you are logged in, go to Settings > System > Licensing.

Content Image

Click on Change license group.

Content Image

Select Free license from the options given and then click on Save.

Content Image

Now, you will be asked to restart Splunk in order for changes to take effect. We will restart Splunk from the terminal itself. Close the current tab. On the terminal, while you are at the /opt/splunk/bin directory, run the following command to restart Splunk:

Command:

./splunk restart
Content Image

Step 6: After Splunk has been restarted, access it again at http://localhost:8000. This time you will not be asked for credentials. You can navigate to Settings > System > Licensing again to verify that you are now on free license. You can also see the daily volume limit here.

Content Image

Step 7: Next, create Splunk index for Sysmon. For this, go to Settings > Data > Indexes.

Content Image

Click on New Index.

Content Image

Enter sysmon as Index Name and then directly click on Save.

Content Image

Step 8: Next, enable Splunk universal forwarding logging. For this, go to Settings > Data > Forwarding and receiving.

Content Image

Click on Add new to Configuring receiving under Receive data.

Content Image

Enter 9997 as the receiving port. And then, click on Save.

Content Image

Task 2: Set up and configure Sysmon and Splunk Universal Forwarder on the Windows machine.

Step 9: On the Windows machine, open Powershell and navigate to the Tools directory present on the Administrator's Desktop. The Sysmon installer is present here. Install Sysmon with the given configuration file using the following command. This will also accept the license if not already accepted.

Command:

./Sysmon64.exe -accepteula -i sysmonconfig.xml
Content Image

Sysmon is now installed. You can find the Sysmon logs in the Event Viewer by navigating to Applications and Services Logs > Microsoft > Windows > Sysmon.

Content Image

Step 10: Next, install the Splunk Universal Forwarder. The installer is present in the Tools directory. But first, you need to know the IP address of the receiving indexer or the Ubuntu machine where Splunk is running.

Command:

ip addr
Content Image

10.0.16.42 is the IP address of the machine where Splunk is running.

Now, you can use the following command on Powershell to quietly install the forwarder.

Command:

msiexec.exe /i splunkforwarder-9.0.3-dd0128b1f8cd-x64-release.msi RECEIVING_INDEXER="10.0.16.42:9997" AGREETOLICENSE=Yes /quiet
Content Image

Wait for a minute before proceeding to the next step.

Step 11: After the forwarder is installed, create an inputs.conf file in C:\Program Files\SplunkUniversalForwarder\etc\system\local\ with the following content:

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = sysmon
sourcetype = "XmlWinEventLog:Microsoft-Windows-Sysmon/Operational"
disabled = false
renderXml = false
Content Image

Save and close the file. Restart the Splunk Universal Forwarder. Ignore any error that you might notice.

Command:

& 'C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe' restart
Content Image

Step 12: Now, switch back to the Ubuntu machine, and go to the Search & Reporting app on Splunk.

Content Image

You can click on Data Summary to view active hosts and other related information.

Content Image

Content Image

Step 13: Now, you can view all the Sysmon events using the following query:

index="sysmon"
This shows all the events generated in the last 24 hours.

Content Image

Task 3: Create and run queries for the following:

Extract events with a particular Event ID
Step 14: The following query can be used to extract events with a particular Event ID say 13:

index="sysmon" EventCode=13
Event ID 13 is fired every time a registry value is set.

Content Image

List all process execution activity
Step 15: All process execution activity can be listed using the following query:

index="sysmon" | stats values(ParentImage) by Image
Content Image

List all commands executed by processes
Step 16: All commands executed can be listed using the following query:

index="sysmon" | stats values(CommandLine) by Image
Content Image

List all commands executed by a specific process
Step 17: To list all commands executed by a specific (usually abused) process, you can use the following query:

index="sysmon" Image="*cmd.exe" | stats values(CommandLine) by Image
Content Image

Identify the longest cmd.exe command that was executed
Step 18: You can identify the longest cmd.exe command that was executed (overly long commands are often suspicious) by using the following query:

index="sysmon" Image="*cmd.exe" | eval len=len(CommandLine) | table len, CommandLine | sort -len
Content Image

Task 4: Run T1518.001-5 (Sysmon Service Discovery) atomic test and detect its activity.

Step 19: First, you need to import the Atomic Red Team (ART) module. The following command imports the Invoke-AtomicTest function each time a new instance of ART is started in PowerShell:

Command:

Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1" -Force
Content Image

Step 20: Let's see the details of the test that we need to run. Use the following command:

Command:

[Atomic test invocation omitted]
Content Image

As per the description, this test will perform discovery of an installed Sysinternals Sysmon service using driver altitude (even if the name is changed). When sucessfully executed, the test is going to display sysmon driver instance if it is installed. The attack command can also be seen here.

Step 21: Run this test.

Command:

[Atomic test invocation omitted]
Content Image

The test was successful.

Step 22: Now, let's try to detect the activity of this test on Splunk. Make sure you are initially viewing all the events first. Now, on the left hand side, you can see list of Selected Fields. Let's add the ParentCommandLine field to it. For this, click on All Fields and simply tick mark the field you want to add.

Content Image

Content Image

Step 23: Once the ParentCommandLine field is added, click on it to view the top 10 values.

Content Image

The command highlighted above looks suspicious. This is the attack command that we performed to check if Sysmon is installed or not using the driver altitude. The fltMC.exe program is a system-supplied command line utility for common minifilter driver management operations primarily used by developers and system administrators. Note that the sysmon altitude is always the same - 385201.

Clicking on it would take us to the matching events. Note the query that is executed.

Content Image

Now, you can expand the events for more information.

Content Image

Task 5: Run T1547.004-5 (Winlogon HKLM Userinit Key Persistence) atomic test and detect its activity.

Winlogon.exe is a Windows component responsible for actions at logon/logoff as well as handling the secure attention sequence. Adversaries may abuse features of Winlogon to execute DLLs and/or executables when a user logs in. Specifically, the Userinit, Shell and Notify subkeys under Winlogon are known to be vulnerable to abuse.

[Operational test reference omitted]

Step 24: Let's see the details of the test that we need to run. Use the following command:

Command:

[Atomic test invocation omitted]
Content Image

The Winlogon Userinit registry key typically points to userinit.exe, the user initialization program that is executed when a user logs on. This test would set Winlogon Userinit key to execute a binary at logon along with userinit.exe. This could be a malicious executable allowing an attacker to gain persistence. However, in this case, it is only cmd.exe.

Step 25: Run this test.

Command:

[Atomic test invocation omitted]
Content Image

The test was successful.

Step 26: Now let's go to Splunk detect the activity of this. Let's try to list all the executed commands associated with the powershell.exe process. Run the following query:

index="sysmon" Image="*powershell.exe" | stats values(CommandLine) by Image
Content Image

We found the exact activity that took place. It is clear that the Winlogon Userinit registry key was modified to execute cmd.exe along with userinit.exe at logon. So, now we know what the target object was. We also know that an event ID of 13 would have been generated as a registry value was set. Let's try to put these in our query to get to the event.

index="sysmon" Image="*powershell.exe" TargetObject="*Userinit" EventCode=13
Content Image

We go the exact event. Expand it to see more information.

Content Image

Task 6: Run T1047-1,2 (WMI Reconnaissance Users and Processes) atomic tests and detect their activity.

Windows Management Instrumentation (WMI) is an administration feature that provides a uniform environment to access Windows system components. An adversary can use WMI to interact with local and remote systems and use it as a means to execute various behaviors, such as gathering information for Discovery as well as remote Execution of files as part of Lateral Movement.

[Operational test reference omitted]

Step 27: Let's see the details of the tests that we need to run one by one. Use the following command:

Command:

[Atomic test invocation omitted]
Content Image

This test will make use of WMI to list all local user accounts. The attack command is also given here. Let's look at test 2.

Command:

[Atomic test invocation omitted]
Content Image

This test will make use of WMI to list running processes. The attack command is also given here.

Step 28: Let's run test 1.

Command:

[Atomic test invocation omitted]
Content Image

The test was successful.

Step 29: Let's run test 2.

Command:

[Atomic test invocation omitted]
Content Image

The test was successful.

Step 30: Let's try to detect these activities on Splunk. There can be mutliple ways to do it. Let's try to use the following query:

index="sysmon" Image="*WMIC.exe*"
WMIC.exe is a built-in Microsoft program that allows command-line access to the WMI. Running this query gives us a bunch of events.

Content Image

Let's see the ParentCommandLine field values from the left navigation.

Content Image

These are some suspicious commands as one tried to gather information regarding the processes and the local users accounts on the system. This could be an attacker.

Clicking on them will take us to their respective events. Let's click on the first value.

Content Image

Expand them for more information.

Content Image

Conclusion
In this lab, we learned how to set up and configure Splunk, Splunk Universal Forwarder and Sysmon. We forwarded Sysmon logs via Splunk Universal Forwarder to Splunk running on the Ubuntu machine. Then we ran basic queries to search through and analyze the data. We also used Atomic Red Team to emulate attacks and attempted to detect the activities of the same on Splunk.

References
Splunk

Atomic Red Team

Symon

Sysmon Modular